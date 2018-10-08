在[第28集]()的async_handshake中使用.

同步io,所有的读写方法都是同步的。****重点****  
` socket -> read_some ` / ` asio::write `
```
template <typename Stream, typename Operation>
std::size_t io(Stream& next_layer, stream_core& core, const Operation& op, asio::error_code& ec)
{
  std::size_t bytes_transferred = 0;
  do switch (op(core.engine_, ec, bytes_transferred))
  {
    //read方法，加个try
  case engine::want_input_and_retry:

    if (core.input_.size() == 0)
      core.input_ = asio::buffer(core.input_buffer_, next_layer.read_some(core.input_buffer_, ec));

    // Pass the new input data to the engine.
    core.input_ = core.engine_.put_input(core.input_);

    // Try the operation again.
    continue;

    //write方法,往socket中写数据
  case engine::want_output_and_retry:
    asio::write(next_layer, core.engine_.get_output(core.output_buffer_), ec);

    // Try the operation again.
    continue;

  case engine::want_output:
    asio::write(next_layer, core.engine_.get_output(core.output_buffer_), ec);

    core.engine_.map_error_code(ec);

    return bytes_transferred;

  default:
    core.engine_.map_error_code(ec);
    return bytes_transferred;

  } while (!ec);

  core.engine_.map_error_code(ec);
  return 0;
}
```

异步是一个类模板，函数类。
```
template <typename Stream, typename Operation, typename Handler>
class io_op
{
  io_op(Stream& next_layer, stream_core& core,
    const Operation& op, Handler& handler)
  : next_layer_(next_layer),
    core_(core),
    op_(op),
    start_(0),
    want_(engine::want_nothing),
    bytes_transferred_(0),
    handler_(ASIO_MOVE_CAST(Handler)(handler))
{
}

void operator()(asio::error_code ec, std::size_t bytes_transferred = ~std::size_t(0), int start = 0)
{
  //此处是赋值操作
  //赋值操作返回值是，所赋的值，比如：
  /*
  int i=10;
  int j;
  int t = (j=i);
  此处t的结果是：10
  */
  switch (start_ = start)
  {
  case 1: // Called after at least one async operation.
    do
    {
      switch (want_ = op_(core_.engine_, ec_, bytes_transferred_))
      {
      case engine::want_input_and_retry:
        if (core_.input_.size() != 0)
        {
          core_.input_ = core_.engine_.put_input(core_.input_);
          continue; //这里再try一下
        }

        // The engine wants more data to be read from input. However, we
        // cannot allow more than one read operation at a time on the
        // underlying transport. The pending_read_ timer's expiry is set to
        // pos_infin if a read is in progress, and neg_infin otherwise.
        //不能多个读操作
        if (core_.expiry(core_.pending_read_) == core_.neg_infin())
        {
          // Prevent other read operations from being started.
          core_.pending_read_.expires_at(core_.pos_infin());

          // Start reading some data from the underlying transport.
          //异步读取数据
          next_layer_.async_read_some(asio::buffer(core_.input_buffer_), ASIO_MOVE_CAST(io_op)(*this));
        }
        else
        {
          // Wait until the current read operation completes.
          //等到当前读取完毕
          core_.pending_read_.async_wait(ASIO_MOVE_CAST(io_op)(*this));
        }

        // Yield control until asynchronous operation completes. Control
        // resumes at the "default:" label below.
        return;

      case engine::want_output_and_retry:
      case engine::want_output:
        //不可以多个同时写
        if (core_.expiry(core_.pending_write_) == core_.neg_infin())
        {
          // Prevent other write operations from being started.
          core_.pending_write_.expires_at(core_.pos_infin());

          // Start writing all the data to the underlying transport.
          asio::async_write(next_layer_,  core_.engine_.get_output(core_.output_buffer_), ASIO_MOVE_CAST(io_op)(*this));
        }
        else
        {
          // Wait until the current write operation completes.
          core_.pending_write_.async_wait(ASIO_MOVE_CAST(io_op)(*this));
        }

        // Yield control until asynchronous operation completes. Control
        // resumes at the "default:" label below.
        return;

      default:

        // The SSL operation is done and we can invoke the handler, but we
        // have to keep in mind that this function might be being called from
        // the async operation's initiating function. In this case we're not
        // allowed to call the handler directly. Instead, issue a zero-sized
        // read so the handler runs "as-if" posted using io_context::post().
        if (start)
        {
          next_layer_.async_read_some(asio::buffer(core_.input_buffer_, 0),ASIO_MOVE_CAST(io_op)(*this));

          // Yield control until asynchronous operation completes. Control
          // resumes at the "default:" label below.
          return;
        }
        else
        {
          // Continue on to run handler directly.
          break;
        }
      } //内部的switch完毕

      //外边switch的default
      default:
      if (bytes_transferred == ~std::size_t(0))
        bytes_transferred = 0; // Timer cancellation, no data transferred.
      else if (!ec_)
        ec_ = ec;

      switch (want_)
      {
      case engine::want_input_and_retry:

        // Add received data to the engine's input.
        core_.input_ = asio::buffer(core_.input_buffer_, bytes_transferred);
        core_.input_ = core_.engine_.put_input(core_.input_);

        // Release any waiting read operations.
        core_.pending_read_.expires_at(core_.neg_infin());

        // Try the operation again.
        continue;

      case engine::want_output_and_retry:

        // Release any waiting write operations.
        core_.pending_write_.expires_at(core_.neg_infin());

        // Try the operation again.
        continue;

      case engine::want_output:

        // Release any waiting write operations.
        core_.pending_write_.expires_at(core_.neg_infin());

        // Fall through to call handler.

      default:

        // Pass the result to the handler.
        op_.call_handler(handler_,core_.engine_.map_error_code(ec_), ec_ ? 0 : bytes_transferred_);

        // Our work here is done.
        return;
      }
    } while (!ec_);

    // 错误代码，返回。
    op_.call_handler(handler_, core_.engine_.map_error_code(ec_), 0);
  }
}

//private:
Stream& next_layer_;
stream_core& core_;
Operation op_;
int start_;
engine::want want_;
asio::error_code ec_;
std::size_t bytes_transferred_;
Handler handler_;
};

```
