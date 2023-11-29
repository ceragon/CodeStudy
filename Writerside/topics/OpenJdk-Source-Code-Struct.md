# OpenJdk 源码中常见结构体

## frame 栈帧

以 x86_64 架构为例，官方的示意如下：

```text
------------------------------ Asm interpreter ----------------------------------------
Layout of asm interpreter frame:
   [expression stack      ] * <- sp
   [monitors              ]   \
    ...                        | monitor block size
   [monitors              ]   /
   [monitor block size    ]
   [byte code pointer     ]                   = bcp()                bcp_offset
   [pointer to locals     ]                   = locals()             locals_offset
   [constant pool cache   ]                   = cache()              cache_offset
   [methodData            ]                   = mdp()                mdx_offset
   [klass of method       ]                   = mirror()             mirror_offset
   [Method*               ]                   = method()             method_offset
   [last sp               ]                   = last_sp()            last_sp_offset
   [old stack pointer     ]                     (sender_sp)          sender_sp_offset
   [old frame pointer     ]   <- fp           = link()
   [return pc             ]
   [oop temp              ]                     (only for native calls)
   [locals and parameters ]
                              <- sender sp
------------------------------ Asm interpreter ----------------------------------------
```

frame 在 x86 架构中的表示:

```c++
class frame {
 private:
    union {
        intptr_t* _sp; // stack pointer (from Thread::last_Java_sp)
        int _offset_sp; // used by frames in stack chunks
    };
    address   _pc; // program counter (the next instruction after the call)
    mutable CodeBlob* _cb;
    mutable const ImmutableOopMap* _oop_map;
    enum deopt_state {
        not_deoptimized,
        is_deoptimized,
        unknown
    };
    deopt_state _deopt_state;
    bool        _on_heap;
    
    enum {
        sender_sp_offset                                 =  2,
        metadata_words                                   = sender_sp_offset,
    }
    union {
        intptr_t*  _fp; // frame pointer
        int _offset_fp; // relative frame pointer for use in stack-chunk frames
    };
    union {
        intptr_t* _unextended_sp;
        int _offset_unextended_sp; // for use in stack-chunk frames
    };
}
```