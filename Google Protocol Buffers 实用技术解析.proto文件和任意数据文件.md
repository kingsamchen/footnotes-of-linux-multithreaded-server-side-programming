# Google Protocol Buffers 实用技术：解析.proto文件和任意数据文件

Google Protocol Buffers 是一种非常方便高效的数据编码方式（data serialization），几乎在Google的每个产品中都用到了。本文介绍 protocol buffers 的一种高级使用方法（在Google Protocol Buffer的主页上没有的）。

Protocol Buffers 通常的使用方式如下：我们的程序只依赖于有限的几个 protocol messages。我们把这几个 message 定义在一个或者多个 .proto 文件里，然后用编译器 protoc 把 .proto 文件翻译成 C++ 语言（.h和.cc文件）。这个翻译过程把每个 message 翻译成了一个 C++ class。这样我们的程序就可以使用这些 protocol classes 了。

但是还有一种不那么常见的使用方式：我们有一批数据文件（或者网络数据流），其中包含了一些 protocol messages 内容。我们也有定义这些 protocol messages 的 .proto 文件。我们希望解析数据文件中的内容，但是不能使用 protoc 编译器。

一个例子是 codex。codex是Google内最常用的一个工具程序。它可以解析任意文件中的 protocol message 内容，并且把这些内容打印成人能方便的阅读的格式。为了能正确解析和打印数据文件内容，codex 需要定义 protocol message 的 .proto 文件。

为了实现 codex 的功能，一种ad hoc的方法是：
1 把 codex 的基本功能（比如读取数据文件，打印文件内容等）实现在一个 .cc 文件里（比如叫做 codex-base.cc)
1 对给定的 .proto 文件，调用 protoc，得到对应的 .pb.h 和 .pb.cc 文件。
1 把 codex-base.cc 和 protoc 的结果一起编译，生成一个专门解析某一个 protocol message 内容的 codex 程序。
这个办法太ad hoc了。它为每个 .protoc 文件编写一个 codex 程序。

另一个办法是，如果我们有世界上所有的 .proto 文件，那么我们把它们都预先用 protoc 编译了，链接进 codex。显然这个搞法也是不现实的。

那么codex到底是怎么实现的呢？其实它利用了 protocol buffers 没有写入文档的一些 API。闲话少说，我们来看一段用这些“神秘的API“写的代码。这段代码用于解析任意给定的 .proto 文件：

```cpp
#include <google/protobuf/descriptor.h>
#include <google/protobuf/dynamic_message.h>
#include <google/protobuf/io/zero_copy_stream_impl.h>
#include <google/protobuf/io/tokenizer.h>
#include <google/protobuf/compiler/parser.h>

//-----------------------------------------------------------------------------
// Parsing given .proto file for Descriptor of the given message (by
// name).  The returned message descriptor can be used with a
// DynamicMessageFactory in order to create prototype message and
// mutable messages.  For example:
/*
DynamicMessageFactory factory;
const Message* prototype_msg = factory.GetPrototype(message_descriptor);
const Message* mutable_msg = prototype_msg->New();
*/
//-----------------------------------------------------------------------------
void GetMessageTypeFromProtoFile(const string& proto_filename,
                              FileDescriptorProto* file_desc_proto) {
using namespace google::protobuf;
using namespace google::protobuf::io;
using namespace google::protobuf::compiler;

FILE* proto_file = fopen(proto_filename.c_str(), "r");
{
 if (proto_file == NULL) {
   LOG(FATAL) << "Cannot open .proto file: " << proto_filename;
 }

 FileInputStream proto_input_stream(fileno(proto_file));
 Tokenizer tokenizer(&proto_input_stream, NULL);
 Parser parser;
 if (!parser.Parse(&tokenizer, file_desc_proto)) {
   LOG(FATAL) << "Cannot parse .proto file:" << proto_filename;
 }
}
fclose(proto_file);

// Here we walk around a bug in protocol buffers that
  // |Parser::Parse| does not set name (.proto filename) in
  // file_desc_proto.
  if (!file_desc_proto->has_name()) {
 file_desc_proto->set_name(proto_filename);
}
}
```

这个函数的输入是一个 .proto 文件的文件名。输出是一个 FileDescriptorProto 对象。这个对象里存储着对 .proto 文件解析之后的结果。我们接下来用这些结果动态生成某个 protocol message 的 instance（或者用C++术语叫做object）。然后可以调用这个 instance 自己的 ParseFromArray/String 成员函数，来解析数据文件中的每一条记录的内容。请看如下代码：

```cpp
//-----------------------------------------------------------------------------
// Print contents of a record file with following format:
//
//   { <int record_size> <KeyValuePair> }
//
// where KeyValuePair is a proto message defined in mpimr.proto, and
// consists of two string fields: key and value, where key will be
// printed as a text string, and value will be parsed into a proto
// message given as |message_descriptor|.
//-----------------------------------------------------------------------------
void PrintDataFile(const string& data_filename,
                const FileDescriptorProto& file_desc_proto,
                const string& message_name) {
const int kMaxRecieveBufferSize = 32 * 1024 * 1024;  // 32MB
  static char buffer[kMaxRecieveBufferSize];

ifstream input_stream(data_filename.c_str());
if (!input_stream.is_open()) {
 LOG(FATAL) << "Cannot open data file: " << data_filename;
}

google::protobuf::DescriptorPool pool;
const google::protobuf::FileDescriptor* file_desc =
 pool.BuildFile(file_desc_proto);
if (file_desc == NULL) {
 LOG(FATAL) << "Cannot get file descriptor from file descriptor"
            << file_desc_proto.DebugString();
}

const google::protobuf::Descriptor* message_desc =
 file_desc->FindMessageTypeByName(message_name);
if (message_desc == NULL) {
 LOG(FATAL) << "Cannot get message descriptor of message: " << message_name;
}

google::protobuf::DynamicMessageFactory factory;
const google::protobuf::Message* prototype_msg =
 factory.GetPrototype(message_desc);
if (prototype_msg == NULL) {
 LOG(FATAL) << "Cannot create prototype message from message descriptor";
}
google::protobuf::Message* mutable_msg = prototype_msg->New();
if (mutable_msg == NULL) {
 LOG(FATAL) << "Failed in prototype_msg->New(); to create mutable message";
}

uint32 proto_msg_size; // uint32 is the type used in reocrd files.
  for (;;) {
 input_stream.read((char*)&proto_msg_size, sizeof(proto_msg_size));

 if (proto_msg_size > kMaxRecieveBufferSize) {
   LOG(FATAL) << "Failed to read a proto message with size = "
              << proto_msg_size
              << ", which is larger than kMaxRecieveBufferSize ("
              << kMaxRecieveBufferSize << ")."
              << "You can modify kMaxRecieveBufferSize defined in "
              << __FILE__;
 }

 input_stream.read(buffer, proto_msg_size);
 if (!input_stream)
   break;

 if (!mutable_msg->ParseFromArray(buffer, proto_msg_size)) {
   LOG(FATAL) << "Failed to parse value in KeyValuePair:" << pair.value();
 }

 cout << mutable_msg->DebugString();
}

delete mutable_msg;
}
```

这个函数需要三个输入：1）数据文件的文件名，2）之前GetMessageTypeFromProtoFile函数返回的FileDescriptorProto对象，3）数据文件中每条记录的对应的protocol message 的名字（注意，一个 .proto 文件里可以定义多个 protocol messages，所以我们需要知道数据记录对应的具体是哪一个 message）。

以上代码中利用了 DescriptorPool 从 FileDescriptorProto 解析出 FileDescriptor（描述 .proto 文件中所有的 messages）。然后用 DynamicMessageFactory 从 FileDescriptor 里找到我们关注的那个 message 的 MessageDescriptor。接下来，我们利用 DynamicMessageFactory 根据 MessageDescriptor 得到一个 prototype message instance。注意，这个 instance 是不能往里面写内容的（immutable）。我们需要调用其 New 成员函数，来生成一个 mutable 的 instance。

有了一个对应数据记录的 message instance，接下来就好办了。我们读取数据文件中的每条记录。注意：此处我们假设数据文件中以此存放了一条记录的长度，然后是记录内容，接下来是第二条记录的长度和内容，以此类推。所以在上述函数中，我们循环的读取记录长度，然后解析记录内容。值得注意的是，解析内容利用的是 mutable message instance 的 ParseFromArrary 函数；它需要知道记录的长度。因此我们必须在数据文件中存储每条记录的长度。

接下来这段程序演示如何调用 GetMessageTypeFromProtoFile 和 PrintDataFile：

```cpp
int main(int argc, char** argv) {
string proto_filename, message_name;
vector<string> data_filenames;
FileDescriptorProto file_desc_proto;

ParseCmdLine(argc, argv, &proto_filename, &message_name, &data_filenames);
GetMessageTypeFromProtoFile(proto_filename, &file_desc_proto);

for (int i = 0; i < data_filenames.size(); ++i) {
 PrintDataFile(data_filenames[i], file_desc_proto, message_name);
}

return 0;
}
```

---
1. 见书 P220
2. 原文链接：https://cxwangyi.blogspot.com/2010/06/google-protocol-buffers-proto.html
