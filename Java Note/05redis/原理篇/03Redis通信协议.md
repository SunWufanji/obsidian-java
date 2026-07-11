# Redis 通信协议

## 3.1 RESP 协议

Redis 采用 **RESP（Redis Serialization Protocol）** 作为客户端与服务端的通信协议。

| 版本 | 说明 |
|------|------|
| RESP（RESP2） | Redis 2.0 成为标准，目前默认使用 |
| RESP3 | Redis 6.0 引入，增加更多数据类型，支持客户端缓存 |

### 数据类型格式

RESP 通过**首字节**区分数据类型：

| 首字节 | 类型 | 示例 |
|--------|------|------|
| `+` | 单行字符串 | `+OK\r\n` |
| `-` | 错误信息 | `-Error message\r\n` |
| `:` | 整数 | `:10\r\n` |
| `$` | 多行字符串（二进制安全） | `$6\r\nfoobar\r\n` |
| `*` | 数组 | `*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n` |

**多行字符串特殊值：**
- `$0\r\n\r\n`：空字符串
- `$-1\r\n`：null（不存在）

**客户端发送命令格式（数组）：**

以 `SET name 张三` 为例：
```
*3\r\n
$3\r\n
SET\r\n
$4\r\n
name\r\n
$6\r\n
张三\r\n
```

---

## 3.2 基于 Socket 手写 Redis 客户端

Redis 基于 TCP 通信，可以用 Java Socket 模拟客户端：

```java
public class Main {

    static Socket s;
    static PrintWriter writer;
    static BufferedReader reader;

    public static void main(String[] args) {
        try {
            // 1. 建立连接
            s = new Socket("127.0.0.1", 6379);
            writer = new PrintWriter(new OutputStreamWriter(s.getOutputStream(), StandardCharsets.UTF_8));
            reader = new BufferedReader(new InputStreamReader(s.getInputStream(), StandardCharsets.UTF_8));

            // 2. 认证
            sendRequest("auth", "123321");
            System.out.println("auth: " + handleResponse());

            // 3. SET
            sendRequest("set", "name", "张三");
            System.out.println("set: " + handleResponse());

            // 4. GET
            sendRequest("get", "name");
            System.out.println("get: " + handleResponse());

            // 5. MGET
            sendRequest("mget", "name", "num", "msg");
            System.out.println("mget: " + handleResponse());

        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (reader != null) reader.close();
                if (writer != null) writer.close();
                if (s != null) s.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    // 按 RESP 格式发送命令
    private static void sendRequest(String... args) {
        writer.println("*" + args.length);
        for (String arg : args) {
            writer.println("$" + arg.getBytes(StandardCharsets.UTF_8).length);
            writer.println(arg);
        }
        writer.flush();
    }

    // 解析 RESP 响应
    private static Object handleResponse() throws IOException {
        int prefix = reader.read();
        switch (prefix) {
            case '+': return reader.readLine();
            case '-': throw new RuntimeException(reader.readLine());
            case ':': return Long.parseLong(reader.readLine());
            case '$':
                int len = Integer.parseInt(reader.readLine());
                if (len == -1) return null;
                if (len == 0) return "";
                return reader.readLine();
            case '*': return readArray();
            default: throw new RuntimeException("未知数据类型: " + (char) prefix);
        }
    }

    private static List<Object> readArray() throws IOException {
        int len = Integer.parseInt(reader.readLine());
        if (len <= 0) return null;
        List<Object> list = new ArrayList<>(len);
        for (int i = 0; i < len; i++) {
            list.add(handleResponse());
        }
        return list;
    }
}
```

---

## 面试简答

**Q: Redis 的 RESP 协议是什么？**

A: RESP（Redis Serialization Protocol）是 Redis 客户端与服务端的通信协议。用首字节区分数据类型：`+` 单行字符串、`-` 错误、`:` 整数、`$` 多行字符串、`*` 数组。客户端发送命令时统一用数组格式，每个元素用 `$长度\r\n内容\r\n` 表示。

**Q: 为什么 Redis 能用 Socket 直接通信？**

A: Redis 基于 TCP 协议，RESP 是纯文本协议，格式简单。只要按照 RESP 格式构造请求并解析响应，任何支持 TCP 的语言都可以直接与 Redis 通信，不依赖特定客户端库。
