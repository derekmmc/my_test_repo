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

// pub trait Transfer {
//     fn write_file(&mut self, path:String) -> io::Result<usize>;
// }

// impl Transfer for TcpStream {
//     fn write_file(&mut self, path:String) -> io::Result<usize> {
//         const BUFFSIZE: usize = 1024 * 20;
//         let mut file = File::open(path).expect("Couldn't open file.");
//         let mut buf = [0; BUFFSIZE];
//         let mut get_buffsize: usize = 0;
//         get_buffsize = file.read(&mut buf).unwrap();
//         loop {
//             if get_buffsize == 0 {
//                 break;
//             }
//             println!("{}", get_buffsize);
//             self.write(&buf[..get_buffsize]).expect("Stream write failed!");
//             self.flush();
//             buf = [0; BUFFSIZE];
//             get_buffsize = file.read(&mut buf).unwrap();
//         }
//         Ok(0)
//     }
// }

fn main() {
    let stream = TcpStream::connect("192.168.22.10:8000").unwrap();
    let mut buffer_stream = BufStream::new( stream );

    let path = Path::new(r#"D:\VirtualBoxMachine\tmp_swap\Windows 7 x64-s008.vmdk"#);

    let start_time = time::SystemTime::now();

    let file = File::open(path).expect("failed to open file");
    let mut reader = BufReader::new(file);

    //file length
    let file_length:u64 = reader.get_ref()
                                .metadata()
                                .expect("failed to get meta data")
                                .len();
    //send file length
    buffer_stream.write(&file_length.to_be_bytes()).expect("failed to send file length");
    buffer_stream.flush().unwrap();

    //read buffer
    let buffsize: usize = 1024 * 32 ;

    //calculate sha1
    let mut sha1_value = sha1::Sha1::new();

    let mut buf = vec![0; buffsize];
    let mut get_buffsize = reader.read(&mut buf).unwrap();
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

    let sha1_bytes = sha1_value.result_str().into_bytes();
    println!("sha1: {} {}", sha1_value.result_str(),sha1_value.result_str().len());
    println!("{:?}", sha1_bytes);
    //send sha1
    buffer_stream.write(&sha1_bytes).expect("failed to send sha1 value");
    buffer_stream.flush().unwrap();

    //recv
    let mut status = [0];
    buffer_stream.read_exact(&mut status).expect("failed to get status of transfering file");

    let end_time = time::SystemTime::now();
    let cost_time = end_time.duration_since( start_time ).unwrap();
    println!("take time : {}s", cost_time.as_secs());
}







---server
use std::net::TcpStream;
use std::fs::File;
use std::io::{Read, Write, BufReader};
use std::path::Path;
use std::time;
use crypto::sha1;
use crypto::digest::Digest;
use bufstream::BufStream;

// pub trait Transfer {
//     fn read_file(&mut self, path:String) -> io::Result<usize>;
// }

// impl Transfer for TcpStream {
//     fn read_file(&mut self, path:String) -> io::Result<usize> {
//     	const BUFFSIZE: usize = 1024 * 20;
// 		let mut file = File::create(path).expect("Couldn't create file.");
// 	    let mut buf = [0; BUFFSIZE];
// 	    println!("memory address: {:p}", &buf);
// 	    let mut get_buffsize: usize = 0;
// 	    get_buffsize = self.read(&mut buf).unwrap();
// 	    loop {
// 	    	if get_buffsize == 0 {
// 	    		break;
// 	    	}
// 	    	file.write(&buf[..get_buffsize]);
// 	    	file.flush();
// 	    	get_buffsize = self.read(&mut buf).unwrap();
// 	    }
// 	    Ok(0)
//     }
// }


fn main() {
	let listener = TcpListener::bind("0.0.0.0:8000").expect("Bind ip failed.");
	for stream in listener.incoming() {
    	let mut buffer_stream = BufStream::new( stream );
	    let path = Path::new(r#"D:\VirtualBoxMachine\tmp_swap\test_recv.vmdk"#);

	    //recv file length
	    let mut length_bytes = [0;8];
	    buffer_stream.read_exact(&mut length_bytes).expect("failed to recv file length");
	    let length = u64::from_be_bytes( length_bytes );

	    let buffsize: usize = 1024 * 32 ;
	    //recv times
    	let recv_times:u64 = (length as f32 / buffsize as f32).ceil() as u64;
	}

}


----
rust-crypto = "^0.2"
bufstream = "0.1.4"
