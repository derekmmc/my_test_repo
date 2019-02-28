# my_test_repo
my first text test here.
add some text to this file.







use crate::libs::openssl_crypto;
use std::fs::{File, OpenOptions};
use std::io;
use std::io::{Read, Write, BufReader, BufWriter};
use std::path::Path;
use std::net::{TcpStream, Shutdown};
use std::net::ToSocketAddrs;

use bufstream::BufStream;

pub struct StreamWorker {
	id: i64,
	tcp_stream: BufStream<TcpStream>,
}

impl StreamWorker {
	pub fn new(id:i64, stream: TcpStream) -> StreamWorker {
		StreamWorker {
			id: id,
			tcp_stream: BufStream::new( stream ),
		}
	}

	pub fn write_text(&mut self, text:&String) -> Result<(), &str> {
		//send text length
		self.tcp_stream.write(&text.len().to_be_bytes()).expect("failed to send text length");
	    self.tcp_stream.flush().unwrap();

	    //buf size
		let buffsize:usize = 1024 * 8;

    	//send sha1
	    self.tcp_stream.write(&openssl_crypto::get_text_sha1(&text).as_bytes()).expect("failed to send text hash value");
	    self.tcp_stream.flush().unwrap();

	    //send text
	    for c in text.as_bytes().chunks(buffsize) {
	    	self.tcp_stream.write(&c).expect("failed to write text chunks");
	    	self.tcp_stream.flush().unwrap();
	    }

	    //recv status
	    let mut status = [0];
	    self.tcp_stream.read_exact(&mut status).expect("failed to get status of transfering text");
	    if status == [0] {
	    	Ok(())
	    } else {
	    	Err("failed to send text")
	    }
	}

	pub fn read_text(&mut self) -> Result<String, &str> {
		//recv text length
	    let mut length_bytes = [0;8];
	    self.tcp_stream.read_exact(&mut length_bytes).expect("failed to recv file length");
	    let length = usize::from_be_bytes( length_bytes );

    	//recv text sha1
    	let mut sha1_bytes = [0;40];
    	self.tcp_stream.read_exact(&mut sha1_bytes).unwrap();

	    let mut text_bytes = Vec::with_capacity(length);

	    //write buffer
	    let buffsize: usize = 1024 * 8;

	    //recv times
    	let recv_times:usize = (length as f32 / buffsize as f32).ceil() as usize;
    	for c in 0..recv_times {

    		let buf_len:usize = if (c+1)*buffsize >= length {
					    			length - c*buffsize
					    		} else {
					    			buffsize
					    		};

    		let mut buf = vec![0;buf_len];
    		self.tcp_stream.read_exact(&mut buf).unwrap();

    		text_bytes.append(&mut buf);
    	}

    	//calculate sha1
    	let text = String::from_utf8(text_bytes).unwrap();

    	//send status
    	let mut status = [0];
    	if sha1_bytes.to_vec() != openssl_crypto::get_text_sha1(&text).as_bytes() {
    		status = [1];
    	}
    	self.tcp_stream.write(&status).unwrap();
    	self.tcp_stream.flush().unwrap();

    	if status == [0] {
    		Ok(text)
    	} else {
    		Err("failed to recv text")
    	}
	}

	pub fn write_file(&mut self, path:&Path) -> Result<(), &str> {
		let file = File::open(path).expect("failed to open file");
	    let mut reader = BufReader::new(file);

	    //file length
	    let file_length:usize = reader.get_ref()
	                                .metadata()
	                                .expect("failed to get meta data")
	                                .len() as usize;
	    //send file length
	    self.tcp_stream.write(&file_length.to_be_bytes()).expect("failed to send file length");
	    self.tcp_stream.flush().unwrap();

	    //write buffer
	    let buffsize: usize = 1024 * 128 ;

	    //calculate sha1
	    let sha1_value = openssl_crypto::get_file_sha1(path);

	    //send file
	    let mut buf = vec![0; buffsize];
	    let mut get_buffsize = reader.read(&mut buf).unwrap();
	    loop {
	        if get_buffsize == 0 {
	            break
	        }
	        self.tcp_stream.write(&buf[..get_buffsize]).expect("failed to write stream");
	        self.tcp_stream.flush().unwrap();
	        for item in buf.iter_mut() {
	            *item = 0;
	        }
	        get_buffsize = reader.read(&mut buf).unwrap();
	    }

	    //calculate sha1
	    let sha1_bytes = sha1_value.as_bytes();
	    //send sha1
	    self.tcp_stream.write(&sha1_bytes).expect("failed to send sha1 value");
	    self.tcp_stream.flush().unwrap();

	    //recv
	    let mut status = [0];
	    self.tcp_stream.read_exact(&mut status).expect("failed to get status of transfering file");

	    match status {
	        [0] => {
	        	Ok(())
	        },
	        _ => {
	        	Err("failed to send file")
	        },
	    }
	}

	pub fn read_file(&mut self, path:&Path) -> Result<(), &str> {
		//file handler
	    let file = OpenOptions::new()
	    							.write(true)
	    							.truncate(true)
	    							.create(true)
	    							.open(path)
	    							.expect("failed to create file");

	    let mut buf_file = BufWriter::new(file);

	    //recv file length
	    let mut length_bytes = [0;8];
	    self.tcp_stream.read_exact(&mut length_bytes).expect("failed to recv file length");
	    let length = usize::from_be_bytes( length_bytes );

	    //write buffer
	    let buffsize: usize = 1024 * 128 ;



	    //recv file
    	let recv_times:usize = (length as f32 / buffsize as f32).ceil() as usize;
    	for c in 0..recv_times {
    		let buf_len:usize = if (c+1)*buffsize >= length {
					    			length - c*buffsize
					    		} else {
					    			buffsize
					    		};

    		let mut buf = vec![0;buf_len];
    		self.tcp_stream.read_exact(&mut buf).unwrap();

			buf_file.write(&buf).unwrap();
			buf_file.flush().unwrap();
    	}

    	//recv sha1
    	let mut sha1_bytes = [0;40];
    	self.tcp_stream.read_exact(&mut sha1_bytes).unwrap();

	    //calculate sha1
    	let sha1_value = openssl_crypto::get_file_sha1(path);
    	//send status
    	let mut status = [0];
    	if sha1_bytes.to_vec() != sha1_value.as_bytes() {
    		status = [1];
    	}
    	self.tcp_stream.write(&status).unwrap();
    	self.tcp_stream.flush().unwrap();

    	if status == [0] {
    		Ok(())
    	} else {
    		Err("failed to recv file")
    	}

	}

	pub fn close(&mut self) -> io::Result<()> {
		self.tcp_stream.get_mut().shutdown(Shutdown::Both)
	}

}

