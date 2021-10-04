---
type: docs
title: "I/O"
linkTitle: "I/O"
weight: 30
---

[原文链接：更好的Scala I/O: better-files](http://colobu.com/2016/05/11/better-files-Simple-safe-and-intuitive-Scala-I-O/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)

## 添加依赖
		
	libraryDependencies += "com.github.pathikrit" %% "better-files" % version

## 实例化

	import better.files._
	import java.io.{File => JFile}
	val f = File("/User/johndoe/Documents")                      // using constructor
	val f1: File = file"/User/johndoe/Documents"                 // using string interpolator
	val f2: File = "/User/johndoe/Documents".toFile              // convert a string path to a file
	val f3: File = new JFile("/User/johndoe/Documents").toScala  // convert a Java file to Scala
	val f4: File = root/"User"/"johndoe"/"Documents"             // using root helper to start from root
	val f5: File = `~` / "Documents"                             // also equivalent to `home / "Documents"`
	val f6: File = "/User"/"johndoe"/"Documents"                 // using file separator DSL
	val f7: File = home/"Documents"/"presentations"/`..`         // Use `..` to navigate up to parent

## 文件读写

	val file = root/"tmp"/"test.txt"
	file.overwrite("hello")
	file.appendLine().append("world")
	assert(file.contentAsString == "hello\nworld")

或者类似 Shell 风格：

	file < "hello"     // same as file.overwrite("hello")
	file << "world"    // same as file.appendLines("world")
	assert(file! == "hello\nworld")

或者：

	"hello" `>:` file
	"world" >>: file
	val bytes: Array[Byte] = file.loadBytes

流式接口风格：

	(root/"tmp"/"diary.txt")
	 .createIfNotExists()  
	 .appendLine()
	 .appendLines("My name is", "Inigo Montoya")
	 .moveTo(home/"Documents")
	 .renameTo("princess_diary.txt")
	 .changeExtensionTo(".md")
	 .lines

## Stream和编码

生成迭代器：

	val bytes  : Iterator[Byte]            = file.bytes
	val chars  : Iterator[Char]            = file.chars
	val lines  : Iterator[String]          = file.lines
	val source : scala.io.BufferedSource   = file.newBufferedSource // needs to be closed, unlike the above APIs which auto closes when iterator ends

编解码：

	val content: String = file.contentAsString  // default codec
	// custom codec:
	import scala.io.Codec
	file.contentAsString(Codec.ISO8859)
	//or
	import scala.io.Codec.string2codec
	file.write("hello world")(codec = "US-ASCII")

## 与Java交互

转换成Java对象：

	val file: File = tmp / "hello.txt"
	val javaFile     : java.io.File                 = file.toJava
	val uri          : java.net.uri                 = file.uri
	val reader       : java.io.BufferedReader       = file.newBufferedReader 
	val outputstream : java.io.OutputStream         = file.newOutputStream 
	val writer       : java.io.BufferedWriter       = file.newBufferedWriter 
	val inputstream  : java.io.InputStream          = file.newInputStream
	val path         : java.nio.file.Path           = file.path
	val fs           : java.nio.file.FileSystem     = file.fileSystem
	val channel      : java.nio.channel.FileChannel = file.newFileChannel
	val ram          : java.io.RandomAccessFile     = file.newRandomAccess
	val fr           : java.io.FileReader           = file.newFileReader
	val fw           : java.io.FileWriter           = file.newFileWriter(append = true)
	val printer      : java.io.PrintWriter          = file.newPrintWriter

以及：

	file1.reader > file2.writer       // pipes a reader to a writer
	System.in > file2.out             // pipes an inputstream to an outputstream
	src.pipeTo(sink)                  // if you don't like symbols
	val bytes   : Iterator[Byte]        = inputstream.bytes
	val bis     : BufferedInputStream   = inputstream.buffered  
	val bos     : BufferedOutputStream  = outputstream.buffered   
	val reader  : InputStreamReader     = inputstream.reader
	val writer  : OutputStreamWriter    = outputstream.writer
	val printer : PrintWriter           = outputstream.printWriter
	val br      : BufferedReader        = reader.buffered
	val bw      : BufferedWriter        = writer.buffered
	val mm      : MappedByteBuffer      = fileChannel.toMappedByteBuffer

## 模式匹配

	/**
	 * @return true if file is a directory with no children or a file with no contents
	 */
	def isEmpty(file: File): Boolean = file match {
	  case File.Type.SymbolicLink(to) => isEmpty(to)  // this must be first case statement if you want to handle symlinks specially; else will follow link
	  case File.Type.Directory(files) => files.isEmpty
	  case File.Type.RegularFile(content) => content.isEmpty
	  case _ => file.notExists    // a file may not be one of the above e.g. UNIX pipes, sockets, devices etc
	}
	// or as extractors on LHS:
	val File.Type.Directory(researchDocs) = home/"Downloads"/"research"

## 通配符

	val dir = "src"/"test"
	val matches: Iterator[File] = dir.glob("**/*.{java,scala}")
	// above code is equivalent to:
	dir.listRecursively.filter(f => f.extension == Some(".java") || f.extension == Some(".scala"))

或者使用正则表达式：
		
	val matches = dir.glob("^\\w*$")(syntax = File.PathMatcherSyntax.regex

## 文件系统操作

	file.touch()
	file.delete()     // unlike the Java API, also works on directories as expected (deletes children recursively)
	file.clear()      // If directory, deletes all children; if file clears contents
	file.renameTo(newName: String)
	file.moveTo(destination)
	file.copyTo(destination)       // unlike the default API, also works on directories (copies recursively)
	file.linkTo(destination)                     // ln file destination
	file.symbolicLinkTo(destination)             // ln -s file destination
	file.{checksum, md5, sha1, sha256, sha512, digest}   // also works for directories
	file.setOwner(user: String)    // chown user file
	file.setGroup(group: String)   // chgrp group file
	Seq(file1, file2) >: file3     // same as cat file1 file2 > file3
	Seq(file1, file2) >>: file3    // same as cat file1 file2 >> file3
	file.isReadLocked / file.isWriteLocked / file.isLocked
	File.newTemporaryDirectory() / File.newTemporaryFile() // create temp dir/file
	
## UNIX DSL

提供了UNIX风格的操作：

	import better.files_, Cmds._   // must import Cmds._ to bring in these utils
	pwd / cwd     // current dir
	cp(file1, file2)
	mv(file1, file2)
	rm(file) /*or*/ del(file)
	ls(file) /*or*/ dir(file)
	ln(file1, file2)     // hard link
	ln_s(file1, file2)   // soft link
	cat(file1)
	cat(file1) >>: file
	touch(file)
	mkdir(file)
	mkdirs(file)         // mkdir -p
	chown(owner, file)
	chgrp(owner, file)
	chmod_+(permission, files)  // add permission
	chmod_-(permission, files)  // remove permission
	md5(file) / sha1(file) / sha256(file) / sha512(file)
	unzip(zipFile)(targetDir)
	zip(file*)(zipFile)

## 文件属性

	file.name       // simpler than java.io.File#getName
	file.extension
	file.contentType
	file.lastModifiedTime     // returns JSR-310 time
	file.owner / file.group
	file.isDirectory / file.isSymbolicLink / file.isRegularFile
	file.isHidden
	file.hide() / file.unhide()
	file.isOwnerExecutable / file.isGroupReadable // etc. see file.permissions
	file.size                 // for a directory, computes the directory size
	file.posixAttributes / file.dosAttributes  // see file.attributes
	file.isEmpty      // true if file has no content (or no children if directory) or does not exist
	file.isParentOf / file.isChildOf / file.isSiblingOf / file.siblings

`chmod`操作：

	import java.nio.file.attribute.PosixFilePermission
	file.addPermission(PosixFilePermission.OWNER_EXECUTE)      // chmod +X file
	file.removePermission(PosixFilePermission.OWNER_WRITE)     // chmod -w file
	assert(file.permissionsAsString == "rw-r--r--")
	// The following are all equivalent:
	assert(file.permissions contains PosixFilePermission.OWNER_EXECUTE)
	assert(file(PosixFilePermission.OWNER_EXECUTE))
	assert(file.isOwnerExecutable)

## 文件比较

	file1 == file2    // equivalent to `file1.isSamePathAs(file2)`
	file1 === file2   // equivalent to `file1.isSameContentAs(file2)` (works for regular-files and directories)
	file1 != file2    // equivalent to `!file1.isSamePathAs(file2)`
	file1 =!= file2   // equivalent to `!file1.isSameContentAs(file2)`

排序操作：

	val files = myDir.list.toSeq
	files.sorted(File.Order.byName) 
	files.max(File.Order.bySize) 
	files.min(File.Order.byDepth) 
	files.max(File.Order.byModificationTime) 
	files.sorted(File.Order.byDirectoriesFirst)

## 解压缩

// Unzipping:
	val zipFile: File = file"path/to/research.zip"
	val research: File = zipFile.unzipTo(destination = home/"Documents"/"research") 
	// Zipping:
	val zipFile: File = directory.zipTo(destination = home/"Desktop"/"toEmail.zip")
	// Zipping/Unzipping to temporary files/directories:
	val someTempZipFile: File = directory.zip()
	val someTempDir: File = zipFile.unzip()
	assert(directory === someTempDir)
	// Gzip handling:
	File("countries.gz").newInputStream.gzipped.lines.take(10).foreach(println)

## 轻量级的ARM (自动化的资源管理)

Auto-close Java closeables:

	for {
	  in <- file1.newInputStream.autoClosed
	  out <- file2.newOutputStream.autoClosed
	} in.pipeTo(out)

`better-files`提供了更加便利的管理，因此下面的代码：

	for {
	 reader <- file.newBufferedReader.autoClosed
	} foo(reader)

可以改写为：

	for {
	 reader <- file.bufferedReader    // returns ManagedResource[BufferedReader]
	} foo(reader)
	// or simply:
	file.bufferedReader.map(foo)

## Scanner

	val data = t1 << s"""
	  | Hello World
	  | 1 true 2 3
	""".stripMargin
	val scanner: Scanner = data.newScanner()
	assert(scanner.next[String] == "Hello")
	assert(scanner.lineNumber == 1)
	assert(scanner.next[String] == "World")
	assert(scanner.next[(Int, Boolean)] == (1, true))
	assert(scanner.tillEndOfLine() == " 2 3")
	assert(!scanner.hasNext)

或者可以写定制的Scanner。

## 文件监控

普通的Java文件监控：

	import java.nio.file.{StandardWatchEventKinds => EventType}
	val service: java.nio.file.WatchService = myDir.newWatchService
	myDir.register(service, events = Seq(EventType.ENTRY_CREATE, EventType.ENTRY_DELETE))

`better-files`抽象了一个更加简单的接口：

	val watcher = new ThreadBackedFileMonitor(myDir, recursive = true) {
	  override def onCreate(file: File) = println(s"$file got created")
	  override def onModify(file: File) = println(s"$file got modified")
	  override def onDelete(file: File) = println(s"$file got deleted")
	}
	watcher.start()

或者使用下面的写法：

	import java.nio.file.{Path, StandardWatchEventKinds => EventType, WatchEvent}
	val watcher = new ThreadBackedFileMonitor(myDir, recursive = true) {
	  override def dispatch(eventType: WatchEvent.Kind[Path], file: File) = eventType match {
	    case EventType.ENTRY_CREATE => println(s"$file got created")
	    case EventType.ENTRY_MODIFY => println(s"$file got modified")
	    case EventType.ENTRY_DELETE => println(s"$file got deleted")
	  }
	}

## 使用Akka进行文件监控

	import akka.actor.{ActorRef, ActorSystem}
	import better.files._, FileWatcher._
	implicit val system = ActorSystem("mySystem")
	val watcher: ActorRef = (home/"Downloads").newWatcher(recursive = true)
	// register partial function for an event
	watcher ! on(EventType.ENTRY_DELETE) {    
	  case file if file.isDirectory => println(s"$file got deleted") 
	}
	// watch for multiple events
	watcher ! when(events = EventType.ENTRY_CREATE, EventType.ENTRY_MODIFY) {   
	  case (EventType.ENTRY_CREATE, file) => println(s"$file got created")
	  case (EventType.ENTRY_MODIFY, file) => println(s"$file got modified")
	}

