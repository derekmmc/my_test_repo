# my_test_repo
my first text test here.
add some text to this file.


















use std::fs;
use std::path::{Path, PathBuf, Component, Prefix};

fn main() {
	// let path_old = "D:\\test\\hello";
	let path = Path::new("E:\\test\\Hello你好");
    // fs::create_dir(path_old).unwrap();
    let mut path_buf = PathBuf::new();
    for c in path.components() {
    	match c {
    		Component::Prefix(prefix_component) => {
    			//println!("get prefix ---> {:?}", c);
    		},
    		Component::RootDir => {
    			//println!("get root_dir ---> {:?}", c);
    		},
    		_=> {
    			let mut path_exists = false;
				let mut new_path = path_buf.clone();
				new_path.push( c );
				println!("{:?}---> {}", new_path, new_path.as_path().exists());
			    // for entry in path_buf.read_dir().expect("read_dir call failed") {
			    // 	if let Ok(entry) = entry {
			    // 		if c.as_os_str().to_str().unwrap().to_lowercase() == entry.file_name().as_os_str().to_str().unwrap().to_lowercase() {
			    // 			if c.as_os_str() != entry.file_name().as_os_str() {
			    // 				println!("rename {:?}--- to --->{:?}", entry.path().as_path(), new_path.as_path() );
			    // 				fs::rename( entry.path().as_path(),new_path.as_path() ).expect("rename failed!");
			    // 				path_exists = true;
			    // 				break;
			    // 			}
			    // 		}
			    // 	}
			    // }
    		}
    	}
    	path_buf.push(c);
    }
}
