# my_test_repo
my first text test here.
add some text to this file.


--client
use std::net::TcpStream;
use std::fs::File;
use std::io::{Read, Write, BufReader};
use std::path::Path;
use std::time;
use crypto::sha1;
use crypto::digest::Digest;
use bufstream::BufStream;

fn main() {
    let stream = TcpStream::connect("127.0.0.1:8000").unwrap();
    let mut buffer_stream = BufStream::new( stream );

    let path = Path::new(r#"C:\test\libcrypto.lib"#);

    let start_time = time::SystemTime::now();

    let file = File::open(path).expect("failed to open file");
    let mut reader = BufReader::new(file);

    //file length
    let file_length:usize = reader.get_ref()
                                .metadata()
                                .expect("failed to get meta data")
                                .len() as usize;
    //send file length
    buffer_stream.write(&file_length.to_be_bytes()).expect("failed to send file length");
    buffer_stream.flush().unwrap();
    println!("send file length.");

    //write buffer
    let buffsize: usize = 1024 * 32 ;

    //calculate sha1
    let mut sha1_value = sha1::Sha1::new();

    let mut buf = vec![0; buffsize];
    let mut get_buffsize = reader.read(&mut buf).unwrap();
    println!("sending file...");
    loop {
        if get_buffsize == 0 {
            break
        }
        //send 
        buffer_stream.write(&buf[..get_buffsize]).expect("failed to write stream");
        buffer_stream.flush().unwrap();
        sha1_value.input( &buf[..get_buffsize] );
        for item in buf.iter_mut() {
            *item = 0;
        }
        //println!("{:?}", &buf[..]);
        get_buffsize = reader.read(&mut buf).unwrap();

        //println!("{:?}", &buf[..]);
    }
    println!("send file finished");

    let sha1_bytes = sha1_value.result_str().into_bytes();
    println!("sha1: {} {}", sha1_value.result_str(),sha1_value.result_str().len());
    println!("{:?}", sha1_bytes);
    //send sha1
    buffer_stream.write(&sha1_bytes).expect("failed to send sha1 value");
    buffer_stream.flush().unwrap();
    println!("send sha1 bytes");
    //recv
    let mut status = [0];
    println!("getting status");
    buffer_stream.read_exact(&mut status).expect("failed to get status of transfering file");

    let end_time = time::SystemTime::now();
    let cost_time = end_time.duration_since( start_time ).unwrap();
    println!("take time : {}s", cost_time.as_secs());
}






---server
use std::net::{TcpListener,TcpStream};
use std::fs::{File, OpenOptions};
use std::io::{Read, Write, BufReader, BufWriter};
use std::path::Path;
use std::time;
use crypto::sha1;
use crypto::digest::Digest;
use bufstream::BufStream;


fn main() {
	let listener = TcpListener::bind("0.0.0.0:8000").expect("Bind ip failed.");
	for stream in listener.incoming() {
    	let mut buf_stream = BufStream::new( stream.unwrap() );
	    let path = Path::new(r#"C:\test\libcrypto111.lib"#);

	    //file handler
	    let file = OpenOptions::new().write(true).create(true).open(path).expect("failed to create file"); 
	    let mut buf_file = BufWriter::new(file);

	    //recv file length
	    let mut length_bytes = [0;8];
	    buf_stream.read_exact(&mut length_bytes).expect("failed to recv file length");
	    let length = usize::from_be_bytes( length_bytes );
	    println!("get file length");

	    //write buffer
	    let buffsize: usize = 1024 * 32 ;
	    let mut buf = vec![0;buffsize];

	    //calculate sha1
    	let mut sha1_value = sha1::Sha1::new();

	    //recv times
	    println!("recving file.");
    	let recv_times:usize = (length as f32 / buffsize as f32).ceil() as usize;
    	for c in 0..recv_times {
    		buf_stream.read_exact(&mut buf).unwrap();
    		let mut end:usize = 0;
    		if (c+1)*buffsize >= length {
    			end = length - c*buffsize;
    		} else {
    			end = buffsize;
    		}
			buf_file.write(&buf[..end]).unwrap();
			buf_file.flush().unwrap();
			sha1_value.input( &buf[..end] );

    		for item in buf.iter_mut() {
    			*item = 0;
    		}
    	}

    	//recv sha1
    	let mut sha1_bytes = [0;40];
    	buf_stream.read_exact(&mut sha1_bytes).unwrap();
    	println!("get sha1 bytes.");
    	println!("{:?}", String::from_utf8(sha1_bytes.to_vec()).unwrap() );
    	println!("{:?}", sha1_value.result_str());

    	let mut status = [0];
    	buf_stream.write(&status).unwrap();
    	buf_stream.flush().unwrap();

	}

}


----
rust-crypto = "^0.2"
bufstream = "0.1.4"
