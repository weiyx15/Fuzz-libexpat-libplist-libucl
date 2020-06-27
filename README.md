# 使用AFL/libfuzzer/honggfuzz对libexpat/libplist/libucl进行模糊测试

## libexpat

[libexpat](https://github.com/libexpat/libexpat)是一个解析XML的C语言库

### libexpat AFL

需要重新编译AFL, config.h最大输入文件大小改为100M（libexpat最大测试用例为34M）

使用libFuzzer的测试驱动编译AFL可执行文件  
[common]  
clang -c -w $AFL_PATH/llvm_mode/afl-llvm-rt.o.c   
[驱动1]   
clang -g -fsanitize=address -fsanitize-coverage=trace-pc-guard xml_parse_fuzzer.c -c  
clang++ -g -fsanitize=leak,address afl_driver.cpp xml_parse_fuzzer.o afl-llvm-rt.o.o libexpat.so.1.6.11 -o xml_parse_fuzzerToAFL  

[create screen session] screen -S libexpat_afl_xml_parse  
afl-fuzz -i ../WEIYX_IN -o ../OUT_XML_PARSE -m none ./xml_parse_fuzzerToAFL  
[entry screen session] screen -r libexpat_afl_xml_parse  
[exit screen session] ctrl + A + D  

[驱动2]  
clang -g -fsanitize=address -fsanitize-coverage=trace-pc-guard xml_parsebuffer_fuzzer.c -c  
clang++ -g -fsanitize=leak,address afl_driver.cpp xml_parsebuffer_fuzzer.o afl-llvm-rt.o.o libexpat.so.1.6.11 -o xml_parsebuffer_fuzzerToAFL   

[create screen session] screen -S libexpat_afl_xml_parsebuffer  
afl-fuzz -i ../WEIYX_IN -o ../OUT_XML_PARSEBUFFER -m none ./xml_parsebuffer_fuzzerToAFL  
[entry screen session] screen -r libexpat_afl_xml_parsebuffer  
[exit screen session] ctrl + A + D  

### libexpat libfuzzer
步骤：修改CMakeLists.txt将EXPAT_BUILD_FUZZERS置为ON，重新编译项目cmake .. && make，得到libfuzzer可执行文件xml_parse_fuzzer_UTF-8和xml_parsebuffer_fuzzer_UTF-8   
查看结果：  
cd /home/weiyuxuan/libexpat/expat/build/fuzz  
vim xml_parsebuffer_fuzzer_UTF-8.log   
vim xml_parse_fuzzer_UTF-8.log  
ps -ux|grep xml_parsebuffer_fuzzer_UTF-8   
ps -ux|grep xml_parse_fuzzer_UTF-8  

### libexpat honggfuzz  
编译（在libfuzzer已经修改了CMakeLists.txt的前提下）：  
mkdir build_honggfuzz && cd build_honggfuzz  
cmake -DCMAKE_C_COMPILER=/usr/local/bin/hfuzz-clang -DCMAKE_CXX_COMPILER=/usr/local/bin/hfuzz-clang++ ..  
make  
运行（二进制在./expat/build_honggfuzz/fuzz目录下）：  
[create screen session] screen -S libexpat_honggfuzz_xml_parsebuffer  
honggfuzz -i ../afl/WEIYX_IN -W OUT_XML_PARSEBUFFER -z -- xml_parsebuffer_fuzzer_UTF-8 ___FILE___  
[create screen session] screen -S libexpat_honggfuzz_xml_parse  
honggfuzz -i ../afl/WEIYX_IN -W OUT_XML_PARSE -z -- xml_parse_fuzzer_UTF-8 ___FILE___  

## libplist

[libplist](https://github.com/libimobiledevice/libplist)是一个解析苹果系统plist文件的C语言库

### libplist libfuzzer
./autogen.sh  
./configure --with-fuzzers  
- 修改fuzz/Makefile添加头文件路径: DEFAULT_INCLUDES = -I. -I$(top_builddir) -I${top_builddir}/include
- 注释掉fuzz/Makefile编译libFuzzer.a相关，直接下载libFuzzer源码编译libFuzzer.a复制到fuzz目录下
- 将所有Makefile中"-fsanitize=address -fsanitize-address-use-after-scope -fsanitize=undefined 

-fsanitize-coverage=trace-pc-guard,trace-cmp,edge"替换为"-fsanitize=address,fuzzer" (除了tools/Makefile和test/Makefile)  
make  
[驱动1-目录]/home/weiyuxuan/libplist/fuzz/libfuzzer/bplist_fuzzer  
[驱动2-目录]/home/weiyuxuan/libplist/fuzz/libfuzzer/xplist_fuzzer  

### libplist honggfuzz  
PROJECT_ROOT: ~/libplist_honggfuzz （从libplist复制一份）  
把所有Makefile的CC = clang 改成 CC = hfuzz-clang, CXX = clang++ 改成 CXX = hfuzz-clang++  
把Makefile里所有libFuzzer.a换成libhfuzz.a  
其他对Makefile的改动同libfuzzer  
make  
[create screen session] screen -S libplist_honggfuzz_xplist  
honggfuzz -i ../test/data_xml -W OUT_XPLIST -z -- xplist_fuzzer ___FILE___  
[create screen session] screen -S libplist_honggfuzz_bplist  
honggfuzz -i ../test/data_bin -W OUT_BPLIST -z -- bplist_fuzzer ___FILE___  

### libplist afl  
修改./test/plist_test.c 和 ./test/plist_cmp.c 将argv读入两个文件改成读入一个文件，另一个文件名写死  
将所有Makefile中的CC = clang 改成 CC = ${AFL_HOME}/afl-clang , CXX = clang++ 改成 CXX = $  {AFL_HOME}/afl-clang++  
make  
afl_root: /home/weiyuxuan/libplist_afl/test/.libs  
[create screen session] screen -S libplist_afl_plist_test  
afl-fuzz -i ../data_xml -o OUT_TEST -m none ./plist_test @@  
[create screen session] screen -S libplist_afl_plist_cmp  
afl-fuzz -i ../data_bin -o OUT_CMP -m none -t 1000 ./plist_cmp @@  

## libucl

[libucl](https://github.com/vstakhov/libucl)是一个解析nginx配置文件的C语言库

### libucl libfuzzer
libfuzzer_root = /home/weiyuxuan/libucl/tests/fuzzers/libfuzzer  
./autogen.sh  
./configure  
修改./Makefile:  
  - CC = clang  
  - CPP = clang++  
  - CFLAGS = -g -fsanitize=address,fuzzer  
  - CPPFLAGS = -g -fsanitize=address,fuzzer  

make  
clang -g -fsanitize=address,fuzzer -I ../../include ucl_add_string_fuzzer.c libfuzzer/libucl_la-ucl_parser.o libfuzzer/libucl.a libfuzzer/libFuzzer.a -o ucl_add_string_fuzzer   
clang -g -fsanitize=address,fuzzer -I ../../include -I ../../src -I ../../uthash ucl_msgpack_fuzzer.c libfuzzer/libucl_la-ucl_parser.o libfuzzer/libucl.a libfuzzer/libFuzzer.a -o ucl_msgpack_fuzzer   

### libucl afl
afl_root = /home/weiyuxuan/libucl/tests/fuzzers/afl  
修改./Makefile:  
  - CC = clang
  - CPP = clang++
  - CFLAGS = -g -fsanitize=address
  - CPPFLAGS = -g -fsanitize=address

make  
clang -c -w $AFL_PATH/llvm_mode/afl-llvm-rt.o.c   
clang -g -fsanitize=leak,address -I ../../include ucl_add_string_fuzzer.c -c  
clang++ -g -fsanitize=leak,address -I ../../include ucl_add_string_fuzzer.o afl/afl_driver.cpp  afl/libucl_la-ucl_parser.o afl/libucl.a afl/afl-llvm-rt.o.o -o ucl_add_string_fuzzer  
clang -g -fsanitize=leak,address -I ../../include -I ../../src -I ../../uthash ucl_msgpack_fuzzer.c -c  
clang++ -g -fsanitize=leak,address -I ../../include ucl_msgpack_fuzzer.o afl/afl_driver.cpp  afl/libucl_la-ucl_parser.o afl/libucl.a afl/afl-llvm-rt.o.o -o ucl_msgpack_fuzzer  
[create screen] screen -S libucl_afl_ucl_add_string_fuzzer  
afl-fuzz -i IN -o OUT -m none ./ucl_add_string_fuzzer  
[create screen] screen -S libucl_afl_ucl_msgpack_fuzzer  
afl-fuzz -i IN -o OUT -m none ./ucl_msgpack_fuzzer  

### libucl honggfuzz
libfuzzer_root = /home/weiyuxuan/libucl/tests/fuzzers/honggfuzz  
修改./Makefile:  
  - CC = hfuzz-clang
  - CPP = hfuzz-clang++
  - CFLAGS = -g -fsanitize=address,fuzzer
  - CPPFLAGS = -g -fsanitize=address,fuzzer

make  
clang -g -fsanitize=address,fuzzer -I ../../include ucl_add_string_fuzzer.c honggfuzz/libucl_la-ucl_parser.o honggfuzz/libucl.a honggfuzz/libhfuzz.a -o ucl_add_string_fuzzer  
clang -g -fsanitize=address,fuzzer -I ../../include -I ../../src -I ../../uthash ucl_msgpack_fuzzer.c honggfuzz/libucl_la-ucl_parser.o honggfuzz/libucl.a honggfuzz/libhfuzz.a -o ucl_msgpack_fuzzer   
[create screen session] screen -S libucl_honggfuzz_ucl_add_string  
honggfuzz -i IN -W OUT -z -- ucl_add_string_fuzzer ___FILE___  
[create screen session] screen -S libucl_honggfuzz_ucl_msgpack  
honggfuzz -i IN -W OUT -z -- ucl_msgpack_fuzzer ___FILE___  

## 附：screen sessions操作
查看所有screen session: `ls /var/run/screen/S-weiyuxuan`