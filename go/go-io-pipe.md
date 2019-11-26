+++
title = "Go 中 io.pipe 的源码浅析"
author = "hao.wu"
github_url = "https://sorrycloud.com"
head_img = ""
created_at = 2019-16-22T01:09:25
updated_at = 2019-16-22T01:15:57
description = "go io.pipe 实现了单通道的读写同步.读者写者问题"
tags = ["go"]
+++

## `io.pipe` 库函数
这个库可以类似于OS中经典的线程同步问题：
读者-写者问题。写者向数据池不断存放数据，读者不断从池中读取数据。
当然，这与经典OS的读者写者有明显的差异，这是对于字节流的操作，不是某个资源的操作。
>以下是源代码库的笔记，源代码地址：[io.pipe源码](https://golang.org/src/io/pipe.go)

```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

// Pipe adapter to connect code expecting an io.Reader
// with code expecting an io.Writer.

package io

import (
	"errors"
	"sync"
	"sync/atomic"
)

// atomicError is a type-safe atomic value for errors.
// We use a struct{ error } to ensure consistent use of a concrete type.
type atomicError struct{ v atomic.Value }

func (a *atomicError) Store(err error) {
	a.v.Store(struct{ error }{err})
}
func (a *atomicError) Load() error {
	err, _ := a.v.Load().(struct{ error })
	return err.error
}

// ErrClosedPipe is the error used for read or write operations on a closed pipe.
var ErrClosedPipe = errors.New("io: read/write on closed pipe")

// A pipe is the shared pipe structure underlying PipeReader and PipeWriter.
type pipe struct {
	wrMu sync.Mutex    // 写入数据需要加锁保护，此时不能读写，因此用Mutex
	wrCh chan []byte   // 存储外界写入的数据，可以看成数据池
	rdCh chan int      // 内部数据的空间容量

	once sync.Once     // 操作仅需要一次执行，调用Once即可
	done chan struct{} // pipe是否已经关闭，注意空结构体channel使用方式
	rerr atomicError   // read错误
	werr atomicError   // write错误
}

// 外部的b读取pipe的数据，返回n是读取的字节数，err是任何可能的错误
// 注意理解Read是外部存储读取pipe内部的数据，这相当于读写模式读者
func (p *pipe) Read(b []byte) (n int, err error) {
	select {
	case <-p.done:  // pipe已经关闭了，读取错误
		return 0, p.readCloseError()
	default:       
	}

	select {
	case bw := <-p.wrCh:    // 读取数据，如果有数据正在写入数据池，则发生阻塞
		nr := copy(b, bw)   // 拷贝数据
		p.rdCh <- nr        // 获取读取数据的字节数
		return nr, nil      // 返回个数，没有错误
	case <-p.done:          // pipe已经关闭了，读取错误
		return 0, p.readCloseError()  // 个数为0，并记录错误
	}
}

func (p *pipe) readCloseError() error {
	rerr := p.rerr.Load()
	if werr := p.werr.Load(); rerr == nil && werr != nil {
		return werr
	}
	return ErrClosedPipe
}

func (p *pipe) CloseRead(err error) error {
	if err == nil {
		err = ErrClosedPipe
	}
	p.rerr.Store(err)
	p.once.Do(func() { close(p.done) })
	return nil
}

// 把外部b中的数据写入内部的存储结构中，返回写入数据的个数，err是任何可能的错误
// 注意write是把外部b的数据写入内部，这相当于读写模式的写者
func (p *pipe) Write(b []byte) (n int, err error) {
	select {
	case <-p.done:  // pipe已经关闭，写入肯定发生错误
		return 0, p.writeCloseError()
	default:
		p.wrMu.Lock()          // 写入数据一定要先加锁，这是为了多线程服务
		defer p.wrMu.Unlock()  // 函数结束后解锁
	}

    // 循环的意义：新来的数据肯定要被读入，之后先写入数据，如果写入的数据量多于一次能读取的量，
    // 则一直循环等待下一次写入，期间可能会有阻塞状态
	for once := true; once || len(b) > 0; once = false {
		select {
		case p.wrCh <- b:    // 写入数据，如果之前的数据没有被读出，则一直处于阻塞状态
			nw := <-p.rdCh   // 内部的容量
			b = b[nw:]       // 返回未写入的数据，第一次写入数据肯定是全部写入
			n += nw          // 记录写入的个数
		case <-p.done:       // 如果pipe关闭，返回已经写入的数据个数和关闭错误
			return n, p.writeCloseError()
		}
	}
	return n, nil
}

func (p *pipe) writeCloseError() error {
	werr := p.werr.Load()
	if rerr := p.rerr.Load(); werr == nil && rerr != nil {
		return rerr
	}
	return ErrClosedPipe
}

func (p *pipe) CloseWrite(err error) error {
	if err == nil {
		err = EOF
	}
	p.werr.Store(err)
	p.once.Do(func() { close(p.done) })  // 关闭一次即可，使用Once进行操作
	return nil
} 

// PipeReader is the read half of a pipe.
// 只实现了pipe的read功能，相当于一个读者
type PipeReader struct {
	p *pipe  // 使用指针，说明是共享数据池
}

// Read implements the standard Read interface:
// it reads data from the pipe, blocking until a writer
// arrives or the write end is closed.
// If the write end is closed with an error, that error is
// returned as err; otherwise err is EOF.
func (r *PipeReader) Read(data []byte) (n int, err error) {
	return r.p.Read(data)
}

// Close closes the reader; subsequent writes to the
// write half of the pipe will return the error ErrClosedPipe.
func (r *PipeReader) Close() error {
	return r.CloseWithError(nil)
}

// CloseWithError closes the reader; subsequent writes
// to the write half of the pipe will return the error err.
func (r *PipeReader) CloseWithError(err error) error {
	return r.p.CloseRead(err)
}

// A PipeWriter is the write half of a pipe.
// 只实现了pipe的writer的功能，相当于写者
type PipeWriter struct {
	p *pipe  // 使用指针，说明是共享数据池
}

// Write implements the standard Write interface:
// it writes data to the pipe, blocking until one or more readers
// have consumed all the data or the read end is closed.
// If the read end is closed with an error, that err is
// returned as err; otherwise err is ErrClosedPipe.
func (w *PipeWriter) Write(data []byte) (n int, err error) {
	return w.p.Write(data)
}

// Close closes the writer; subsequent reads from the
// read half of the pipe will return no bytes and EOF.
func (w *PipeWriter) Close() error {
	return w.CloseWithError(nil)
}

// CloseWithError closes the writer; subsequent reads from the
// read half of the pipe will return no bytes and the error err,
// or EOF if err is nil.
//
// CloseWithError always returns nil.
func (w *PipeWriter) CloseWithError(err error) error {
	return w.p.CloseWrite(err)
}

// Pipe creates a synchronous in-memory pipe.
// It can be used to connect code expecting an io.Reader
// with code expecting an io.Writer.
//
// Reads and Writes on the pipe are matched one to one
// except when multiple Reads are needed to consume a single Write.
// That is, each Write to the PipeWriter blocks until it has satisfied
// one or more Reads from the PipeReader that fully consume
// the written data.
// The data is copied directly from the Write to the corresponding
// Read (or Reads); there is no internal buffering.
//
// It is safe to call Read and Write in parallel with each other or with Close.
// Parallel calls to Read and parallel calls to Write are also safe:
// the individual calls will be gated sequentially.
func Pipe() (*PipeReader, *PipeWriter) {
	p := &pipe{
		wrCh: make(chan []byte),
		rdCh: make(chan int),
		done: make(chan struct{}),
	}
	return &PipeReader{p}, &PipeWriter{p}
}
```