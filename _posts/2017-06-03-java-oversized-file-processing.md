---
layout: post
title: Java 超大文件处理
---

**TL;DR**   利用 Memory Mapped File 并发处理超大文件

现需要处理一批超大文件（size > 5G），文件内容是字符串格式的数据，每行为一条数据。如果使用简单的 FileInput 会造成内存溢出。

对于这样的大文件我们有三种方式来处理：
```java
// 1. 使用BufferedInputStream
BufferedInputStream fileIn = new BufferedInputStream(new FileInputStream(fileName), arraySize);
fileIn.read();
```

```java
// 2. 使用FileChannel
FileInputStream fileIn = new FileInputStream(fileName);
FileChannel fileChannel = fileIn.getChannel();
fileChannel.read();
```

```java
// 3. 使用Memory Mapped File
FileInputStream fileIn = new FileInputStream(fileName);
FileChannel fileChannel = fileIn.getChannel();
MappedByteBuffer  mappedBuf = fileChannel.map(FileChannel.MapMode.READ_ONLY, 0, fileLength);
mappedBuf.get();
```

这里我利用 map 将文件映射到内存，并对其分块，并利用多线程并发处理这些映射的文件分块，由于各个线程处理的数据是独立的，所以这就是多线程处理模式中的 Immutable 模式。
```java
public void parallelReadFile(final String path) throws IOException {
    /******************************************************************************************
     * calculate partitions size to map file
     ******************************************************************************************/
    long K_BYTE = 1024L;            // 1KB
    long M_BYTE = 1024 * 1024L;     // 1MB
    long defaultPartitionSize = 50 * M_BYTE;

    List<Long> partitions = new ArrayList<>();

    RandomAccessFile fra = new RandomAccessFile(path, "r");
    FileChannel fc_fra = fra.getChannel();
    long fileSize = fc_fra.size();

    long offset = 0;
    while ((offset + defaultPartitionSize) < fileSize) {
        offset += defaultPartitionSize;
        fc_fra.position(offset);

        Scanner scanner = new Scanner(fc_fra);
        String line = scanner.nextLine();

        partitions.add(defaultPartitionSize + line.length());
        offset += line.length();
    }

    partitions.add(fileSize - offset);
    fc_fra.close();
    fra.close();

    /******************************************************************************************
     * map file & parse in threads
     ******************************************************************************************/
    ExecutorService threadPool = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    FileInputStream fis = new FileInputStream(path);
    FileChannel fc_fis = fis.getChannel();

    offset = 0;
    for (Long size : partitions) {
        if (size > defaultPartitionSize * 2) {
            continue;
        }

        final long areaFrom = offset;
        final long areaTo = offset + size;
        final MappedByteBuffer byteBuffer = fc_fis.map(FileChannel.MapMode.READ_ONLY, offset, size);
        offset += size;

        threadPool.execute(() -> {
            try {
                Charset charset = Charset.forName("iso-8859-1");
                CharsetDecoder decoder = charset.newDecoder();
                CharBuffer charBuffer = decoder.decode(byteBuffer);
                Scanner scanner = new Scanner(charBuffer);
                while (scanner.hasNextLine()) {
                    String line = scanner.nextLine();
                    //TODO: parse `line`
                }
            } catch (CharacterCodingException e) {
                logger.error(e.getMessage(), e);
            }
        });
    }

    /******************************************************************************************
     * wait until thread pool terminaled
     ******************************************************************************************/
    threadPool.shutdown();
    try {
        while (!threadPool.isTerminated()) {
            threadPool.awaitTermination(Long.MAX_VALUE, TimeUnit.SECONDS);
        }
    } catch (InterruptedException e) {
        logger.error(e.getMessage(), e);
    }

    fc_fis.close();
    fis.close();
}
```
