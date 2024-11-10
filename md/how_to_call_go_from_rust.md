# How to call go from rust

## step

### 1. Write go code
```go
package main

/*
#include <stdlib.h>
*/
import "C"

import (
	"fmt"
	"unsafe"

	"github.com/codewithtoucans/goffi/base"
)

//export Greet
func Greet(name *C.char) *C.char {
	return C.CString(fmt.Sprintf(base.Hello, C.GoString(name)))
}

//export GoFree
func GoFree(ptr *C.char) {
	C.free(unsafe.Pointer(ptr))
}

func main() {}
```

### 2. Perpare in rust
#### We need bindgen to c code which go generated auto bind rust
```toml
[build-dependencies]
bindgen = "0.70"
```

#### In rust root dir crate a build.rs
```rust
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]

use std::env;
use std::path::PathBuf;
use std::process::Command;

fn main() {
    let out_path = PathBuf::from(env::var("OUT_DIR").unwrap());

    let mut go_build = Command::new("go");
    go_build
        .current_dir("./src-go")
        .arg("build")
        .arg("-buildmode=c-archive")
        .arg("-o")
        .arg(out_path.join("libgo.a"))
        .arg("./lib.go");

    go_build.status().expect("Go build failed");

    let bindings = bindgen::Builder::default()
        .header(out_path.join("libgo.h").to_str().unwrap())
        .parse_callbacks(Box::new(bindgen::CargoCallbacks::new()))
        .generate()
        .expect("Unable to generate bindings");

    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("Couldn't write bindings!");

    println!("cargo:rerun-if-changed=src-go");
    println!("cargo:rustc-link-search=native={}", out_path.display());
    println!("cargo:rustc-link-lib=static=go");
}
```

#### Let rust project include generate rust code
```rust
use std::ffi::{c_char, CStr, CString};

include!(concat!(env!("OUT_DIR"), "/bindings.rs"));

pub fn greet(name: &str) -> String {
    let name = CString::new(name).unwrap();
    unsafe {
        let cstr = CStr::from_ptr(Greet(name.as_ptr() as *mut c_char));
        let s = String::from_utf8_lossy(cstr.to_bytes()).to_string();
        GoFree(Greet(name.as_ptr() as *mut c_char));
        s
    }
}
```
Then we can call greet in rust, but the implement is by golang
