---
title: C++/Rust元编程之BrainFuck编译器（constexpr/过程宏解法）
toc: true
original: true
date: 2020-11-06 22:42:40
tags:
- C/C++
- Rust
categories:
- 编程
---

## 引子
接上一篇[C++ 元编程之 BrainFuck 编译器（模板元解法）](https://netcan.github.io/2020/11/04/C-%E5%85%83%E7%BC%96%E7%A8%8B%E4%B9%8BBrainFuck%E7%BC%96%E8%AF%91%E5%99%A8%EF%BC%88%E6%A8%A1%E6%9D%BF%E5%85%83%E8%A7%A3%E6%B3%95%EF%BC%89/)挖了个坑：用constexpr方式实现，我发现更容易实现了，代码不到100行搞定，同时也尝试了一下用Rust过程宏来做元编程，最后我会对这两者进行比较。

之前模板元方式解法不支持嵌套循环，同时也不支持输入输出，在这次实现中，支持嵌套循环、输出。

C++版本：
```c
// compile time
constexpr auto res = brain_fuck(R"(
    ++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.
    >---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.
)");
puts(res);

// runtime
if (argc > 1) puts(brain_fuck(argv[1]));
```

Rust版本：
```rust
println!("{}", brain_fuck!(
    ++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.
    >---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.
));
```

而两者背后实现的算法是一致的。

## C++ constexpr解法
其实模板元解法和constexpr解法能力相同，只是实现代价不同，后者更容易实现，写起来就像普通函数一样。运行结果可看：[https://godbolt.org/z/EYn7PG](https://godbolt.org/z/EYn7PG)

首先定义一个Stream类，用于存放输出结果：

```cpp
template<size_t N>
class Stream {
public:
    constexpr void push(char c) { data_[idx_++] = c; }
    constexpr operator const char*() const { return data_; }
    constexpr size_t size() { return idx_; }
private:
    size_t idx_{};
    char data_[N]{};
};
```

然后写一个parse函数，解析BrainFuck代码，经典的递归下降分析：
```cpp
template<typename STREAM>
constexpr auto parse(const char* input, bool skip, char* cells,
        size_t& pc, STREAM&& output) -> size_t {
    const char* c = input;
    while(*c) {
        switch(*c) {
            case '+': if (!skip) ++cells[pc]; break;
            case '-': if (!skip) --cells[pc]; break;
            case '.': if (!skip) output.push(cells[pc]); break;
            case '>': if (!skip) ++pc; break;
            case '<': if (!skip) --pc; break;
            case '[': {
                while (!skip && cells[pc] != 0)
                    parse(c + 1, false, cells, pc, output);
                c += parse(c + 1, true, cells, pc, output) + 1;
            } break;
            case ']': return c - input;
            default: break;
        }
        ++c;
    }
    return c - input;
}

constexpr size_t CELL_SIZE = 16;
template<typename STREAM>
constexpr auto parse(const char* input, STREAM&& output) -> STREAM&& {
    char cells[CELL_SIZE]{};
    size_t pc{};
    parse(input, false, cells, pc, output);
    return output;
}
```

最后用brain_fuck函数串起来：
```cpp
template<size_t OUTPUT_SIZE = 15>
constexpr auto brain_fuck(const char* input) {
    Stream<OUTPUT_SIZE> output;
    return parse(input, output);
}
```

以上就是实现一个BrainFuck编译器的所有细节。

延伸一下，如果你细心的话，你会发现输出大小需要手动指定（默认15字节），如果大小过大，那么多余的空间浪费了；如果大小过小，编译报错。思考一下，有什么办法确定大小呢？毕竟C++20之前constexpr不支持动态分配内存，像链表这种随时扩容的方式暂时不可行。

我们可以实现一个函数brain_fuck_output_size来提前计算好所需要大小：
```cpp
// calculate output size
constexpr auto brain_fuck_output_size(const char* input) -> size_t {
    struct {
        size_t sz{};
        constexpr void push(...) { ++sz; }
    } dummy;
    return parse(input, dummy).sz + 1; // include '\0'
}
```

我们实现一个dummy对象，其push接口只是简单地计数，最终dummy的长度就是输出的长度。

这也是为啥STREAM作为模板参数类型的原因，因为只需要依赖`push`接口，而不需要依赖具体的类型，这也是泛型的魅力。

```c
#define BRAIN_FUCK(in) brain_fuck<brain_fuck_output_size(in)>(in)
constexpr auto res = BRAIN_FUCK(R"(
    ++++++++[>++++[>++>+++>+++>+<<<<-]>+>+>->>+[<]<-]>>.
    >---.+++++++..+++.>>.<-.<.+++.------.--------.>>+.>++.
)");
```

## Rust过程宏解法
Rust做元编程，目前只能通过宏的方式做，而且能力也有限。这里需要用过程宏手段。目前我把`brain_fuck`提交到cargo仓库了：[https://crates.io/crates/brain_fuck](https://crates.io/crates/brain_fuck)，可以体验一下。

同样地写一个parse函数：
```rust
const CELL_SIZE: usize = 16;
fn parse(code: &[u8], skip: bool, cells: &mut [u8; CELL_SIZE],
    pc: &mut usize, output: &mut Vec<u8>) -> usize {
    let mut idx = 0;
    while idx < code.len() {
        let c = code[idx];
        match c {
            b'+' if !skip => cells[*pc] += 1,
            b'-' if !skip => cells[*pc] -= 1,
            b'.' if !skip => output.push(cells[*pc]),
            b'>' if !skip => *pc += 1,
            b'<' if !skip => *pc -= 1,
            b'[' => {
                while !skip && cells[*pc] != 0 {
                    parse(&code[idx+1..], false, cells, pc, output);
                }
                idx += parse(&code[idx+1..], true, cells, pc, output) + 1;
            },
            b']' => return idx,
            _ => {}
        }
        idx += 1;
    }
    idx
}
```

用过程宏brain_fuck包装一下：
```rust
#[proc_macro]
pub fn brain_fuck(_item: TokenStream) -> TokenStream {
    let input = _item.to_string();
    let mut cells: [u8; CELL_SIZE] = [0; CELL_SIZE];
    let mut pc = 0;
    let mut output = Vec::<u8>::new();

    parse(&input.as_bytes(), false, &mut cells, &mut pc, &mut output);

    TokenStream::from_str(
        &format!("\"{}\"", from_utf8(&output).unwrap())
    ).unwrap()
}
```

## 结论
对比C++和Rust版本，实现代价一样。

C++版本实现过程中可以先不加constexpr关键字，通过打印等debug手段调试通过后，最终加上constexpr关键字即可，最后既可以在运行时使用，也可以在编译时使用。如果在编译期出现内存越界（cells越界）情况下，编译报错，即避免了ub。

Rust实现过程宏只能通过lib方式做，同样地也可以直接加打印，在编译的时候输出，最终将打印去掉。输出结果可以直接用`Vec`这种动态容器存，C++20之前暂时得通过定长（预留长度或提前计算）数组搞。而Rust的过程宏只能用在编译时，无法用在运行时，而且只支持字面量方式，不支持变量传参给过程宏。

而本质上Rust宏是在预处理期做替换，而不是在编译期，能做的事有限。例如过程宏无法组合，实现如下效果：
```rust
decode!(encode!("hello world"));
```

而C++的constexpr函数是可以随意组合的，这点只能期待Rust的const fn发力了。

从生成的汇编结果来看，C++版本更加简单粗暴，g++编译器生成的汇编字符串结果直接存到8字节整型中，clang则比较直观，main和数据只有15行：
```nasm
main:                                   # @main
        subq    $24, %rsp
        movq    .L__const.main.res+16(%rip), %rax
        movq    %rax, 16(%rsp)
        movups  .L__const.main.res(%rip), %xmm0
        movaps  %xmm0, (%rsp)
        leaq    8(%rsp), %rdi
        callq   puts
        xorl    %eax, %eax
        addq    $24, %rsp
        retq
.L__const.main.res:
        .quad   13                              # 0xd
        .asciz  "Hello World!\n\000"
        .zero   1
```

而Rust编译器生成的汇编结果就不够C++那么简洁紧凑，这里就不贴出来了。

