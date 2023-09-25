---
layout: post
title: Hello, Rust!
---

The majority of my professional experience is with the C programming language.
However, one year ago I've joined Cisco where I've started working on a [FUSE
filesystem](https://github.com/project-machine/puzzlefs) implemented in Rust.

I've never seen a line of Rust code at this point and I remember thinking how
strange the syntax looked. Needless to say, I've felt completely lost,
wondering what I got myself into. However, if I had any chance of working on
this project, I *had* to learn Rust.

I soon found [a bug](https://github.com/project-machine/puzzlefs/pull/33) and
eagerly tried to fix it, but the borrow checker wasn't impressed about my
enthusiasm. I just wanted a mutable reference to the first element of the
vector and then removing it further down below with
[remove](https://doc.rust-lang.org/stable/std/vec/struct.Vec.html#method.remove),
but the compiler wasn't having it!
Code snippet:
```
fn merge_chunks(
    chunks: &mut Vec<FileChunk>,
    files: &mut Vec<File>,
    prev_files: &mut Vec<File>,
) -> io::Result<()> {
    let mut file = &mut prev_files[0];
    let mut file_used: u64 = Iterator::sum(file.chunk_list.chunks.iter().map(|c| c.len));

    for current_chunk in chunks.drain(..) {
        let mut chunk_used = 0;
        eprintln!("current chunk: {:?}", current_chunk);
        while chunk_used < current_chunk.len {
            eprintln!("md.len {:?}, file_used {:?}, chunk_len: {:?}, chunk_used {:?}", file.md.len(), file_used, current_chunk.len, chunk_used);
            let room = min(file.md.len() - file_used, current_chunk.len - chunk_used);

            let blob = BlobRef {
                offset: chunk_used,
                kind: current_chunk.blob.kind,
            };

            file.chunk_list.chunks.push(FileChunk { blob, len: room });

            chunk_used += room;
            file_used += room;

            // get next file
            if file_used == file.md.len() {
                files.push(prev_files.remove(0));
                if prev_files.is_empty() {
                    break;
                }
                file = &mut prev_files[0];
                file_used = Iterator::sum(file.chunk_list.chunks.iter().map(|c| c.len));
            }
        }
    }
    Ok(())
}
```
And the errors:
```

error[E0499]: cannot borrow `*prev_files` as mutable more than once at a time
   --> builder/src/lib.rs:128:28
    |
106 |     let mut file = &mut prev_files[0];
    |                         ---------- first mutable borrow occurs here
...
113 |             eprintln!("md.len {:?}, file_used {:?}, chunk_len: {:?}, chunk_used {:?}", file.md.len(), file_used, current_chunk.len, chunk...
    |                                                                                        ------------- first borrow later used here
...
128 |                 files.push(prev_files.remove(0));
    |                            ^^^^^^^^^^^^^^^^^^^^ second mutable borrow occurs here

error[E0502]: cannot borrow `*prev_files` as immutable because it is also borrowed as mutable
   --> builder/src/lib.rs:129:20
    |
106 |     let mut file = &mut prev_files[0];
    |                         ---------- mutable borrow occurs here
...
113 |             eprintln!("md.len {:?}, file_used {:?}, chunk_len: {:?}, chunk_used {:?}", file.md.len(), file_used, current_chunk.len, chunk...
    |                                                                                        ------------- mutable borrow later used here
...
129 |                 if prev_files.is_empty() {
    |                    ^^^^^^^^^^^^^^^^^^^^^ immutable borrow occurs here

warning: variable does not need to be mutable
   --> builder/src/lib.rs:356:21
    |
356 |                 let mut file = File {
    |                     ----^^^^
    |                     |
    |                     help: remove this `mut`
    |
    = note: `#[warn(unused_mut)]` on by default

Some errors have detailed explanations: E0499, E0502.
For more information about an error, try `rustc --explain E0499`.
warning: `builder` (lib) generated 1 warning
error: could not compile `builder` due to 2 previous errors; 1 warning emitted
```

Reading stackoverlow answers wasn't much help because I had no idea about the
concepts they were mentioning. So I had to do something that every programmer
fears: read the [documentation](https://doc.rust-lang.org/book/).

The Rust book was quite a nice read, it took me a while to finish it but
afterwards I've finally understood why my first attempt at solving the issue
didn't work: once I had a mutable reference to any element in the vector, the
borrow checker prevented me from using the vector in any way until I was done
using the mutable reference. I've solved the issue by [removing the first
element from the
vector](https://github.com/project-machine/puzzlefs/pull/33/files#diff-56433df5bc6639997363face15c8e9e58ff3ca3cf5956ba84dc350fdce074588R90),
processing it and then [inserting it back into the
vector](https://github.com/project-machine/puzzlefs/pull/33/files#diff-56433df5bc6639997363face15c8e9e58ff3ca3cf5956ba84dc350fdce074588R140)
if further processing needed to be done.
