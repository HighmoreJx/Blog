# Objective-C 属性知识整理
## 本质
## Atomicity
### atomic(default)
Return a valid value, not junk memory
Does not give you is any sort of guarantee about which of those value you might get. 
Not equal to thread-safe

### Nonatomic 
Try to read in the middle of a write. You could get back garbage data.

## Access
Readonly, readwrite(default)
if you want one property that works as readwrite to you, but read only to everybody else. In .h file, you would declare it as readonly, but in your implementation file, you would redefine it as readwrite

## Storage
### strong(default)
### weak
gives you a reference so you still ‘talk’ to that object, but you are not keeping it alive.

### assign
for primitives.
If you do not have an object, you can not use strong, because strong tells the compiler how to work with pointers.
### copy
It works really well with all kinds of mutable objects.


