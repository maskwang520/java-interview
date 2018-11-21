## Netty拆包器分析
#### ByteToMessageDecoder分析
```java
public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
            ByteBuf buffer;
            if (cumulation.writerIndex() <= cumulation.maxCapacity() - in.readableBytes() && cumulation.refCnt() <= 1) {
                buffer = cumulation;
            } else {
                //复制扩容
                buffer = ByteToMessageDecoder.expandCumulation(alloc, cumulation, in.readableBytes());
            }
            //把新来的数据写入到cumulation中
            buffer.writeBytes(in);
            in.release();
            return buffer;
        }
```

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof ByteBuf) {
            RecyclableArrayList out = RecyclableArrayList.newInstance();
            boolean var10 = false;

            try {
                var10 = true;
                ByteBuf data = (ByteBuf)msg;
                //是否是第一次
                this.first = this.cumulation == null;
                if (this.first) {
                    this.cumulation = data;
                } else {
                    //不是第一次，则调用上述cumulate
                    this.cumulation = this.cumulator.cumulate(ctx.alloc(), this.cumulation, data);
                }

                this.callDecode(ctx, this.cumulation, out);
                var10 = false;
            } catch (DecoderException var11) {
                throw var11;
            } catch (Throwable var12) {
                throw new DecoderException(var12);
            } finally {
                if (var10) {
                    if (this.cumulation != null && !this.cumulation.isReadable()) {
                        this.numReads = 0;
                        this.cumulation.release();
                        this.cumulation = null;
                    } else if (++this.numReads >= this.discardAfterReads) {
                        this.numReads = 0;
                        this.discardSomeReadBytes();
                    }

                    int size = out.size();
                    this.decodeWasNull = !out.insertSinceRecycled();
                    fireChannelRead(ctx, out, size);
                    out.recycle();
                }
            }
            //累加器没有剩下的数据，则把cumulation清理掉
            if (this.cumulation != null && !this.cumulation.isReadable()) {
                this.numReads = 0;
                this.cumulation.release();
                this.cumulation = null;
            } else if (++this.numReads >= this.discardAfterReads) {
                //超过16次，则压缩
                this.numReads = 0;
                this.discardSomeReadBytes();
            }

            int size = out.size();
            this.decodeWasNull = !out.insertSinceRecycled();
            //传递解码好的包发动给下一个handler
            fireChannelRead(ctx, out, size);
            out.recycle();
        } else {
            ctx.fireChannelRead(msg);
        }

    }
```

```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        try {
            while(true) {
                if (in.isReadable()) {
                    int outSize = out.size();
                    //如果可读，则触发read的事件
                    if (outSize > 0) {
                        fireChannelRead(ctx, out, outSize);
                        out.clear();
                        outSize = 0;
                    }
                    //可读的字节数
                    int oldInputLength = in.readableBytes();
                    //调用用户decode的逻辑
                    this.decode(ctx, in, out);
                    if (!ctx.isRemoved()) {
                        if (outSize == out.size()) {
                            //读取了部分数据，还要继续
                            if (oldInputLength != in.readableBytes()) {
                                continue;
                            }
                        } else {
                            //list增加了，但是ByteBuf里面的数据并没有减少，则抛异常
                            if (oldInputLength == in.readableBytes()) {
                                throw new DecoderException(StringUtil.simpleClassName(this.getClass()) + ".decode() did not read anything but decoded a message.");
                            }
                            //如果不是一次性的解码，则继续
                            if (!this.isSingleDecode()) {
                                continue;
                            }
                        }
                    }
                }
                //调用结束
                return;
            }
        } catch (DecoderException var6) {
            throw var6;
        } catch (Throwable var7) {
            throw new DecoderException(var7);
        }
    }
```