# my_test_repo
my first text test here.
add some text to this file.


--client
use std::path::Path;
use std::net::TcpStream;
use std::time;

use bufstream::BufStream;

use cfm_tools::libs::db_op;
use cfm_tools::libs::openssl_crypto;
use cfm_tools::libs::network;

use std::thread;


mod cmds;

fn main() {
	//db_op::init_dbs();
	// db_op::hello();
	// cmds::add::hello();
	//
	// let s = time::SystemTime::now();
	//
	// let path = Path::new(r#"D:\tmp\transfer\abc.mp4"#);
	// println!("{}", openssl_crypto::get_file_sha1(&path));
	//
	// println!("{}", time::SystemTime::now().duration_since( s ).unwrap().subsec_millis() );

	let path = String::from(r#"D:\tmp\transfer\abc.mp4"#);
	println!("{}",path);

	let mut connected_id = 0;

	let stream = TcpStream::connect("127.0.0.1:7000").unwrap();
	let mut stream_worker = network::StreamWorker::new( connected_id, stream );

	let s = time::SystemTime::now();

	let mut count = 0;
	loop {

		if count > 10 {
			break;
		}

		let path = String::from( "E:\\tmp\\abc.mp4" );
		let sending_text = String::from( format!("Sending-->{}", count) );
		let end_text = String::from( format!("End-->{}", count) );

		stream_worker.write_text(&sending_text).expect("failed to write text.");
		println!("Send --> {}", count);

		stream_worker.write_file( &path ).expect("failed to write file.");

		stream_worker.write_text(&end_text).expect("failed to write text 2.");
		println!("End ---> {}", count);

		count += 1;

	}
	println!("take time : {}s", time::SystemTime::now().duration_since( s ).unwrap().as_secs() );

}


--server
//use cfm_tools::libs::thread_pool::ThreadPool;
use std::net::{TcpListener, TcpStream};
use std::path::Path;

use bufstream::BufStream;

use cfm_tools::libs::network;

use std::time;


fn main() {
	let listen_ip = "0.0.0.0:7000";
	let mut connected_id = 0;

	let listener = TcpListener::bind(listen_ip).expect("Bind ip failed.");

	println!("Server ready!");

	for stream in listener.incoming() {

		let s = time::SystemTime::now();

		let mut stream_worker = network::StreamWorker::new( connected_id, stream.unwrap() );
		connected_id += 1;
		println!("--------->{}", stream_worker.get_id());
		println!("--------->{}", stream_worker.get_id());
		println!("--------->{}", stream_worker.get_id());

		let mut count = 0;

		loop {
			if count > 10 {
				break;
			}
			let file_name = format!("E:\\tmp\\abc_{}.mp4", count );
			println!("{}", stream_worker.read_text().unwrap() );
			stream_worker.read_file( &file_name ).expect("failed to get file");
			println!("{}", stream_worker.read_text().unwrap() );
			count += 1;
		}
		println!("{}", time::SystemTime::now().duration_since( s ).unwrap().as_secs() );
	}

}


--network

use crate::libs::openssl_crypto;
use std::fs::{File, OpenOptions};
use std::io::{Read, Write, BufReader, BufWriter, Result, Error, ErrorKind};
use std::path::Path;
use std::net::{TcpStream, Shutdown};

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

	pub fn get_id(&self) -> i64 {
		self.id
	}

	pub fn write_text(&mut self, text:&String) -> Result<()> {
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
	    	Err( Error::new(ErrorKind::UnexpectedEof, "failed to send file") )
	    }
	}

	pub fn read_text(&mut self) -> Result<String> {
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
    		Err( Error::new(ErrorKind::UnexpectedEof, "failed to recv text") )
    	}
	}

	pub fn write_file(&mut self, file_path:&String) -> Result<()> {
		let path = Path::new(file_path);
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
	        	Err( Error::new(ErrorKind::UnexpectedEof, "failed to send file") )
	        },
	    }
	}

	pub fn read_file(&mut self, file_path:&String) -> Result<()> {
		//file handler
		let path = Path::new(file_path);
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
    		Err( Error::new(ErrorKind::UnexpectedEof, "failed to recv file") )
    	}

	}

	pub fn close(&mut self) -> Result<()> {
		self.tcp_stream.get_mut().shutdown(Shutdown::Both)
	}

}


--lib

use std::fs::{File, OpenOptions};
use std::io::{Read, Write, BufReader, BufWriter};
use std::path::Path;
use std::net::{TcpStream, Shutdown};
use bufstream::BufStream;
use std::str;

use pyo3::prelude::*;
use pyo3::exceptions;
use pyo3::types::PyString;
mod openssl_crypto;

#[pyclass]
pub struct StreamWorker {
	tcp_stream: BufStream<TcpStream>,
}

#[pymethods]
impl StreamWorker {
    #[new]
	pub fn new( obj:&PyRawObject, ip_port: &str ) {
        obj.init(
            StreamWorker {
                tcp_stream: BufStream::new( TcpStream::connect(ip_port).unwrap() ),
            }
        )
	}

	pub fn write_text(&mut self, text:&str) -> PyResult<()> {
		//send text length
        let text = text.to_string();
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
	    	//Err( Error::new(ErrorKind::UnexpectedEof, "failed to send file") )
            Err( PyErr::new::<exceptions::TypeError, _>("failed to send file") )
	    }
	}

	pub fn read_text(&mut self) -> PyResult<String> {
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
    		//Err( Error::new(ErrorKind::UnexpectedEof, "failed to recv text") )
            Err( PyErr::new::<exceptions::TypeError, _>("failed to recv text") )
    	}
	}

	pub fn write_file(&mut self, file_path:&str) -> PyResult<()> {
		let path = Path::new(file_path);
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
	        	//Err( Error::new(ErrorKind::UnexpectedEof, "failed to send file") )
                Err( PyErr::new::<exceptions::TypeError, _>("failed to send file") )
	        },
	    }
	}

	pub fn read_file(&mut self, file_path:&str) -> PyResult<()> {
		//file handler
		let path = Path::new(file_path);
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
    		//Err( Error::new(ErrorKind::UnexpectedEof, "failed to recv file") )
            Err( PyErr::new::<exceptions::TypeError, _>("failed to recv file") )
    	}

	}

	pub fn close(&mut self) -> PyResult<()> {
		match self.tcp_stream.get_mut().shutdown(Shutdown::Both) {
            Ok(()) => Ok(()),
            Err(_) => Err( PyErr::new::<exceptions::TypeError, _>("failed to close connection!") )
        }
	}

}

#[pymodule]
fn scpy(_py: Python<'_>, m: &PyModule) ->PyResult<()> {
    m.add_class::<StreamWorker>()?;
    Ok(())
}


--cargo
[lib]
name = "scpy"
crate-type = ["cdylib"]

[dependencies]
bufstream = "0.1.4"
openssl = { version = "0.10", features = ["vendored"] }
hex = "0.3.2"
dirs = "1.0.4"
time = "0.1.40"
rusqlite = { version = "0.16.0", features = ["bundled"] }
pyo3 = { git="https://github.com/PyO3/pyo3", features=["extension-module"]}
