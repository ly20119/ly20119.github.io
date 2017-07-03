---
layout: default
title: 手动解析previous_gtids_log_event
---

## {{ page.title }}

最近在做的事情，需要解析MySQL binlog中的previous_gtids，首先想到两种方法：
1. google搜现成开源的解析代码；
2. 移植MySQL内核中关于previous_gtid解析的代码；

移植MySQL代码这个事情，本身工作量比较大，而且MySQL中关于GTID的类以及文件有好多，我只是需要一个解析previous_gtids_log_event的工具，没必要搞这么重，所以方法2是下下策。
然后很自然的去google搜索开源解析代码，本以为解析previous_gtids_log_event这种事情，网上应该有很多现成的代码可以拷贝，结果google不但没有找到，而且连previous_gtids_log_event的格式相关的说明都没有搜到（可能是我不会用google）。方法一行不通。

没办法，只能硬着头皮看MySQL源码中关于previous_gtids_log_event中相关的解析方法。
在MySQL中，所有的event都有一个公共的基类log_event，这个基类有个read_event函数，在解析event的时候，通过read_event函数读出一个event的数据，然后根据头部中的type字段，强制类型转换为对应type的event，好像也没有涉及到解析的过程。这可咋办？
源码中不可能没有线索，所以耐着性子看代码，终于发现一个线索：在log_event.cc文件中定义previous_gtids_log_event成员函数的地方，找到了get_str()函数。这个20行左右的函数作用是将previous_gtids_log_event以指定的格式转换为字符串。

```cpp
char *Previous_gtids_log_event::get_str(
  size_t *length_p, const Gtid_set::String_format *string_format) const
{
  DBUG_ENTER("Previous_gtids_log_event::get_str(size_t *)");
  Sid_map sid_map(NULL);
  Gtid_set set(&sid_map, NULL);
  DBUG_PRINT("info", ("temp_buf=%p buf=%p", temp_buf, buf));
  //将previous_gtids_log_event原始数据转为mysql内部数据结构，这里就是解析编码后的数据
  if (set.add_gtid_encoding(buf, buf_size) != RETURN_STATUS_OK)
    DBUG_RETURN(NULL);
  set.dbug_print("set");
  size_t length= set.get_string_length(string_format);
  DBUG_PRINT("info", ("string length= %lu", (ulong) length));
  char* str= (char *)my_malloc(length + 1, MYF(MY_WME));
  if (str != NULL)
  {
    set.to_string(str, string_format);
    if (length_p != NULL)
      *length_p= length;
  }
  DBUG_RETURN(str);
}
```

继续跟进到add_gtid_encoding函数，可以看到，在Gtid_set类中的add_gtid_encoding函数其实就是解析了previous_gtids_log_event：

```cpp
enum_return_status Gtid_set::add_gtid_encoding(const uchar *encoded,
                                               size_t length,
                                               size_t *actual_length)
{
  DBUG_ENTER("Gtid_set::add_gtid_encoding(const uchar *, size_t)");
  ......
  size_t pos= 0;
  uint64 n_sids;
  ......
  // read number of SIDs
  if (length < 8)
  {
    DBUG_PRINT("error", ("(length=%lu) < 8", (ulong) length));
    goto report_error;
  }
  n_sids= uint8korr(encoded);//获取uuid的个数
  pos+= 8;
  // iterate over SIDs
  for (uint i= 0; i < n_sids; i++)//遍历每一个uuid
  {
    // read SID and number of intervals
    if (length - pos < 16 + 8)//长度判断
    {
      DBUG_PRINT("error", ("(length=%lu) - (pos=%lu) < 16 + 8. "
                           "[n_sids=%llu i=%u]",
                           (ulong) length, (ulong) pos, n_sids, i));
      goto report_error;
    }
    rpl_sid sid;
    sid.copy_from(encoded + pos);
    pos+= 16;
    uint64 n_intervals= uint8korr(encoded + pos);//获取该uuid下，gtid区间的个数
    pos+= 8;
    ......
    // iterate over intervals
    if (length - pos < 2 * 8 * n_intervals)//长度判断
    {
      DBUG_PRINT("error", ("(length=%lu) - (pos=%lu) < 2 * 8 * (n_intervals=%llu)",
                           (ulong) length, (ulong) pos, n_intervals));
      goto report_error;
    }
    Interval_iterator ivit(this, sidno);
    rpl_gno last= 0;
    for (uint i= 0; i < n_intervals; i++)//遍历每一个gtid区间，注意：区间是左闭右开的 [start, end)
    {
      // read one interval
      rpl_gno start= sint8korr(encoded + pos);//解析区间的开始gtid
      pos+= 8;
      rpl_gno end= sint8korr(encoded + pos);//解析区间的结束gtid
      pos+= 8;
      if (start <= last || end <= start)//校验区间
      {
        DBUG_PRINT("error", ("last=%lld start=%lld end=%lld",
                             last, start, end));
        goto report_error;
      }
      last= end;
      .....
    }
  }
  .....
}
```

根据以上代码，很容易得到previous_gtid的格式如下：【event body格式，省略了前面19字节的common header】

```
+-------+--------+----------+-------------+--------------+---------------+-------------------------
|sid_num|sid1    |interv_num|interv1_start|interv1_end   |interv2_start  |interv2_end |sid2    |...
|8 bytes|16 bytes|8 bytes   |8 bytes      |8 bytes       |8 bytes        |8 bytes     |16 bytes|...
+-------+--------+----------+-------------+--------------+---------------+-------------------------
```

previous_gtids_log_event格式中，post_header长度为0， 所以除去前面19字节的common header, 剩余的body部分就是要解析的数据部分。
将mysql代码转为手动解析的代码如下：
```cpp
......
int64_t ev_body_len = ev->ev_length_ - BINLOG_EVENT_MIN_HEAD_LEN;
unsigned char* buf = new unsigned char[ev_body_len + 1];
int32_t rc = fread(buf, 1, ev_body_len, fd_);
......
// read number of SIDs
if (ev_body_len < 8)
{
    //error process
}

//get n_sids
decode_uint64_t(buf, &n_sids);//解析uint_64编码
buf_pos += 8;

previous_gtids_.clear();
for (uint64_t i=0; i<n_sids; i++) {
    if (ev_body_len - buf_pos < 16 + 8){
        //error process
    }

    //get sid
    memcpy(sid, buf + buf_pos, ENCODED_SID_LENGTH);
    buf_pos += 16;
    char tmpbuf[37];
    sid_to_string(sid, tmpbuf);
    uuid.assign(tmpbuf);
    previous_gtids_.append(uuid + ":");

    //get n_intervals
    decode_uint64_t(buf + buf_pos, &n_intervals);
    buf_pos += 8;

    if (ev_body_len - buf_pos < 2 * 8 * n_intervals) {
        //error process
    }

    //iterate over intervals
    last = 0;
    for (uint64_t j=0; j<n_intervals; j++) {
        //read one interval: [start, end)
        //get start
        decode_int64_t(buf + buf_pos, &start);
        buf_pos += 8;

        //save start to previous gtid set
        gno_to_string(start, gno_start);
        previous_gtids_.append(gno_start);

        //get end
        decode_int64_t(buf + buf_pos, &end);      
        buf_pos += 8;

        //check valid
        if (start <= last || end <= start) {
            //error process
        }
        last = end;

        if (start < end - 1) {
            //save end to previous gtid set
            gno_to_string(end - 1, gno_end);
            previous_gtids_.append("-" + gno_end);
        }

        if (j < n_intervals - 1) {
            previous_gtids_.append(":");
        }
    }

    if (i < n_sids - 1) {
        previous_gtids_.append(",\n");
    }
}
```

## 靠谱参考：
http://ikarishinjieva.github.io/blog/blog/2014/04/17/PREVIOUS_GTIDS_LOG_EVENT/

{{ page.date | date: "%Y-%m-%d %H:%M:%S" }}
