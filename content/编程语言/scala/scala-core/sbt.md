---
type: docs
title: "SBT"
linkTitle: "SBT"
weight: 28
---

## 子项目构建

### 基本配置文件

首先编辑`project`目录下的`build.properties`和`plugins.sbt`文件：

```scala
// project/build.properties
sbt.version = 0.13.11

// project/plugins.sbt
logLevel := Level.Warn
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.2")
addSbtPlugin("com.typesafe.sbt" % "sbt-native-packager" % "1.0.0")
addSbtPlugin("com.typesafe.sbt" % "sbt-scalariform" % "1.3.0")
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.5.1")
```

### 项目通用配置

项目配置相关的文件均位于`project/`路径，创建新的`CommonSettings.scala`，别写整个项目的基本配置，包括代码风格配置、依赖仓库配置、依赖冲突配置等：

```scala
import sbt._
import Keys._
import sbtassembly.AssemblyPlugin.autoImport._
import sbtassembly.PathList
import com.typesafe.sbt.SbtScalariform.{ScalariformKeys, scalariformSettings}
import scalariform.formatter.preferences._

object CommonSettings {
  	// 代码风格配置 
  	val customeScalariformSettings = ScalariformKeys.preferences := ScalariformKeys.preferences.value
    	.setPreference(AlignSingleLineCaseStatements, true)
    	.setPreference(AlignSingleLineCaseStatements.MaxArrowIndent, 200)
    	.setPreference(AlignParameters, true)
    	.setPreference(DoubleIndentClassDeclaration, true)
    	.setPreference(PreserveDanglingCloseParenthesis, true)
    // 基本配置与仓库
    val settings: Seq[Def.Setting[_]] = scalariformSettings ++ customeScalariformSettings ++ Seq(
      organization := "com.promisehook.bdp",
      scalaVersion := "2.11.8",
      scalacOptions := Seq("-feature", "-unchecked", "-deprecation", "-encoding", "utf8"),
      updateOptions := updateOptions.value.withCachedResolution(true),
      fork in run := true,
      test in assembly := {},
      resolvers += Opts.resolver.mavenLocalFile,
      resolvers ++= Seq(
        DefaultMavenRepository,
        Resolver.defaultLocal,
        Resolver.mavenLocal,
        Resolver.jcenterRepo,
        Classpaths.sbtPluginReleases,
        "scalaz-bintray" at "http://dl.bintray.com/scalaz/releases",
        "Atlassian Releases" at "https://maven.atlassian.com/public/",
        "Apache Staging" at "https://repository.apache.org/content/repositories/staging/",
        "Typesafe repository" at "https://dl.bintray.com/typesafe/maven-releases/",
        "Sonatype OSS Snapshots" at "https://oss.sonatype.org/content/repositories/snapshots",
        "Java.net Maven2 Repository" at "http://download.java.net/maven/2/",
        "softprops-maven" at "http://dl.bintray.com/content/softprops/maven",
        "OpenIMAJ maven releases repository" at "http://maven.openimaj.org",
        "Sonatype OSS Snapshots" at "https://oss.sonatype.org/content/repositories/snapshots",
        "Eclipse repositories" at "https://repo.eclipse.org/service/local/repositories/egit-releases/content/"
      )
    )
    // 依赖冲突合并配置
    val commonAssemblyMergeStrategy = assemblyMergeStrategy in assembly := {
      case PathList("org", "ansj", xs @ _*)                   => MergeStrategy.first
      case PathList("org", "joda", xs @ _*)                   => MergeStrategy.first
      case PathList("org", "apache", xs @ _*)                 => MergeStrategy.first
      case PathList("org", "nlpcn", xs @ _*)                  => MergeStrategy.first
      case PathList("org", "w3c", xs @ _*)                    => MergeStrategy.first
      case PathList("org", "xml", xs @ _*)                    => MergeStrategy.first
      case PathList("javax", "xml", xs @ _*)                  => MergeStrategy.first
      case PathList("edu", "stanford", xs @ _*)               => MergeStrategy.first
      case PathList("org", "cyberneko", xs @ _*)              => MergeStrategy.first
      case PathList("org", "xmlpull", xs @ _*)              => MergeStrategy.first
      case PathList("org", "objenesis", xs @ _*)              => MergeStrategy.first
      case PathList("com", "esotericsoftware", xs @ _*)        => MergeStrategy.first
      case PathList(ps @ _*) if ps.last endsWith ".dic"       => MergeStrategy.first
      case PathList(ps @ _*) if ps.last endsWith ".data"      => MergeStrategy.first
      //  case "application.conf"                             => MergeStrategy.concat
      //  case "unwanted.txt"                                 => MergeStrategy.discard
      case x =>
        val oldStrategy = (assemblyMergeStrategy in assembly).value
        oldStrategy(x)
    }
}
```

### 子项目依赖配置

为各个子项目编写不同的依赖配置：

```scala
// project/CommonDependencies.scala
object CommonDependencies{
  val specsVersion = "3.6.6"
  val specs = Seq(
    "specs2-core", "specs2-junit", "specs2-mock").
    map("org.specs2" %% _ % specsVersion % Test)
  val jodaTime = "joda-time"  % "joda-time" % "2.8.2"
  val PlayJson = "com.typesafe.play" % "play-json_2.11" % "2.5.2"
  val commonDependencies: Seq[ModuleID] = specs ++ Seq(jodaTime, PlayJson)
}
```

### 编写主配置文件

编写`build.sbt`文件：

```scala
name := "Root-Project-Name"

version := "1.0"

scalaVersion := "2.11.8"

lazy val common = project.
  settings(Commons.settings: _*).
  settings(libraryDependencies ++= Dependencies.databaseDependencies).
  settings(libraryDependencies ++= Dependencies.commonDependencies)
  
lazy val webserver = project.
  dependsOn(common).
  settings(Commons.settings: _*).
  settings(libraryDependencies ++= Seq(specs2,filters,evolutions)).	// Play插件
  settings(libraryDependencies ++= Dependencies.akkaDependencies).
  settings(libraryDependencies ++= Dependencies.playDependencies).
  enablePlugins(PlayScala)
  
lazy val proserver = project.
  dependsOn(common).settings(CommonSettings.settings: _*).
  settings(libraryDependencies ++= Dependencies.akkaDependencies).
  settings(libraryDependencies ++= Dependencies.processDependencies).
  settings(CommonSettings.commonAssemblyMergeStrategy)		// 合并依赖冲突
```

此时，在主项目路径运行`sbt -> compile`即可生成子项目目录，同样，可以在各个子项目的目录中添加需要的配置。

