+++
title = "FFI之Rust与C的错误转换"
date = 2026-04-02
+++

## 动机

如果需要C调用Rust程序，在返回错误时，需要对rust的丰富的错误类型转为C能理解的简单数字或者字符串数组类型。

比如： 
- 对于rust中的*简单枚举类型*，可以转换为c中的简单*数值错误码*
- 对于rust中带*参数的枚举错误类型*，可以编写一个函数返回C中的*char数组字符串表示*

## 简单枚举类型

```rust
enum DatabaseError {
  IsReadOnly = 1,
  IOError = 2,
  FileCorrupted = 3,
}

impl From<DatabaseError> for libc::c_int {
    fn from(value: DatabaseError) -> Self {
        match value {
            DatabaseError::IsReadOnly => 1,
            DatabaseError::IOError => 2,
            DatabaseError::FileCorrupted => 3,
        }
    }
}

```

上面rust代码定义了一个简单的枚举类型DatabaseError，为了能够转换为c语言可以识别的错误码，我们实现一个From trait，这个trait定义转换函数from，将rust中的DatabaseError类型转换为c中的int类型。

## 带参数的错误枚举类型
对于这类rust枚举错误类型，光返回一个C错误码已经不够了，还得知道具体的错误信息，这时可以通过定义一个根据rust错误枚举类型，返回C错误字符串描述的函数来实现，比如下面代码

```rust
pub mod errors {
    pub enum DatabaseError {
        IsReadOnly,
        IOError(std::io::Error),
        FileCorrupted(String),
    }

    // 需要.clone().into()用法，转移了所有权
    impl From<DatabaseError> for libc::c_int {
        fn from(value: DatabaseError) -> Self {
            match value {
                DatabaseError::IsReadOnly => 1,
                DatabaseError::IOError(_) => 2,
                DatabaseError::FileCorrupted(_) => 3,
            }
        }
    }

    // 只读，避免.clone().into()
    impl From<&DatabaseError> for libc::c_int {
        fn from(value: &DatabaseError) -> Self {
            match value {
                DatabaseError::IsReadOnly => 1,
                DatabaseError::IOError(_) => 2,
                DatabaseError::FileCorrupted(_) => 3,
            }
        }
    }
}

pub mod c_api {
    use std::ptr;
    use super::errors;

    fn string_to_c(s: &str) -> Option<ptr::NonNull<libc::c_char>> {
        let bytes = s.as_bytes();
        let len = bytes.len();

        unsafe {
            // 分配 len + 1 字节（包含 '\0'）
            let raw: *mut u8 = libc::malloc(len + 1) as *mut u8;
            let buffer = ptr::NonNull::new(raw)?;

            // 拷贝内容
            raw.copy_from_nonoverlapping(bytes.as_ptr(), len);

            // 写入 '\0'
            raw.add(len).write(0_u8);

            Some(buffer.cast())
        }
    }

    #[unsafe(no_mangle)]
    pub extern "C" fn db_error_description(e: Option<&errors::DatabaseError>) -> Option<ptr::NonNull<libc::c_char>> {
        let error = e?;

        let error_str = match error {
            errors::DatabaseError::IsReadOnly => format!("cannot write to read-only database"),
            errors::DatabaseError::IOError(error) => format!("IO ERROR: {error}"),
            errors::DatabaseError::FileCorrupted(s) => format!("File corrupted, run repair: {s}"),
        };
        let c_error = string_to_c(&error_str);
        c_error
    }

    #[unsafe(no_mangle)]
    pub extern "C" fn db_free_string(ptr: *mut libc::c_char) {
        if !ptr.is_null() {
            unsafe {
                libc::free(ptr.cast());
            }
        }
    }

}
```

上面使用libc::malloc和libc::free手动管理内存，更好的方式是使用CString

```rust
pub mod c_safe_api {
    use super::errors;
    use std::ffi::{CString, c_char};
    use std::ptr;

    // 将逻辑封装，处理可能的 Nul 错误
    fn string_to_c_safe(s: &str) -> *mut c_char {
        // 如果字符串中间有 \0，CString::new 会报错
        // 这里用 unwrap_or 或处理错误，防止 C 端拿到乱码
        CString::new(s)
            .map(|c_str| c_str.into_raw())
            .unwrap_or(ptr::null_mut())
    }

    #[unsafe(no_mangle)]
    pub extern "C" fn db_error_description(e: Option<&errors::DatabaseError>) -> *mut c_char {
        let Some(error) = e else {
            return ptr::null_mut();
        };

        let msg = match error {
            errors::DatabaseError::IsReadOnly => "cannot write to read-only database".to_string(),
            errors::DatabaseError::IOError(err) => format!("IO ERROR: {err}"),
            errors::DatabaseError::FileCorrupted(s) => format!("File corrupted, run repair: {s}"),
        };

        string_to_c_safe(&msg)
    }

    #[unsafe(no_mangle)]
    pub unsafe extern "C" fn db_free_string(ptr: *mut c_char) {
        if !ptr.is_null() {
            // SAFETY: 这里的指针必须是之前通过 CString::into_raw 生成的
            unsafe { drop(CString::from_raw(ptr)) };
        }
    }
}
```

对于自定义的错误类型，可以使用#[repr(C)]进行对齐，比如
```rust
// rust中的错误类型
struct ParseError {
    expected: char,
    line: u32,
    ch: u16,
}

impl ParseError {
    /* ... */
}

// C语言对应的结构体
#[repr(C)]
pub struct parse_error {
    pub expected: libc::c_char,
    pub line: u32,
    pub ch: u16,
}

impl From<ParseError> for parse_error {
    fn from(e: ParseError) -> parse_error {
        // 解构
        let ParseError { expected, line, ch } = e;
        // 转换为c的结构体
        parse_error { expected, line, ch }
    }
}
```