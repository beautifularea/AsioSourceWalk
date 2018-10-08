参考[第13集]()

```
struct scheduler_thread_info : public thread_info_base
{
  op_queue<scheduler_operation> private_op_queue;
  long private_outstanding_work;
};
```

看thread_info_base实现，没看懂？？？
```
thread_info_base
```
