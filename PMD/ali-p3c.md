##本地git commit之前进行代码扫描
1.下载阿里巴巴p3c插件，打包
```
git clone https://github.com/alibaba/p3c.git
cd p3c/p3c-pmd
mvn clean package
```
2.然后在会target目录下生成对应的jar，我们此处使用`p3c-pmd-2.0.1-jar-with-dependencies.jar`这个可以直接运行的jar。
最后我们就可以使用这个jar来进行代码扫码了。
java -cp p3c-pmd-2.0.1-jar-with-dependencies.jar net.sourceforge.pmd.PMD  -d Test.java -R rulesets/java/ali-comment.xml,rulesets/java/ali-concurrent.xml,rulesets/java/ali-constant.xml,rulesets/java/ali-exception.xml,rulesets/java/ali-flowcontrol.xml,rulesets/java/ali-naming.xml,rulesets/java/ali-oop.xml,rulesets/java/ali-orm.xml,rulesets/java/ali-other.xml,rulesets/java/ali-set.xml -f text -cache ./pmd-cache
```
Usage: pmd [options]
  Options:
    -failOnViolation, --failOnViolation
      By default PMD exits with status 4 if violations are found. Disable this
      option with '-failOnViolation false' to exit with 0 instead and just
      write the report.
      Default: true
    -auxclasspath
      Specifies the classpath for libraries used by the source code. This is
      used by the type resolution. Alternatively, a 'file://' URL to a text
      file containing path elements on consecutive lines can be specified.
    -benchmark, -b
      Benchmark mode - output a benchmark report upon completion; default to
      System.err.
      Default: false
    -cache
      Specify the location of the cache file for incremental analysis. This
      should be the full path to the file, including the desired file name
      (not just the parent directory). If the file doesn't exist, it will be
      created on the first run. The file will be overwritten on each run with
      the most up-to-date rule violations.
    -dir, -d
      Root directory for sources.
    -encoding, -e
      Specifies the character set encoding of the source code files PMD is
      reading (i.e., UTF-8).
      Default: UTF-8
    -filelist
      Path to a file containing a list of files to analyze.
    -format, -f
      Report format type.
      Default: text
    -help, -h, -H
      Display help on usage.
    -ignorelist
      Path to a file containing a list of files to ignore.
    -language, -l
      Specify a language PMD should use.
    -minimumpriority, -min
      Rule priority threshold; rules with lower priority than configured here
      won't be used. Valid values are integers between 1 and 5 (inclusive),
      with 5 being the lowest priority.
      Default: 5
    -no-cache
      Explicitly disable incremental analysis. The '-cache' option is ignored
      if this switch is present in the command line.
      Default: false
    -norulesetcompatibility
      Disable the ruleset compatibility filter. The filter is active by
      default and tries automatically 'fix' old ruleset files with old rule
      names
      Default: false
    -property, -P
      {name}={value}: Define a property for the report format.
      Default: []
    -reportfile, -r
      Sends report output to a file; default to System.out.
  * -rulesets, -R
      Comma separated list of ruleset names to use.
    -shortnames
      Prints shortened filenames in the report.
      Default: false
    -showsuppressed
      Report should show suppressed rule violations.
      Default: false
    -stress, -S
      Performs a stress test.
      Default: false
    -suppressmarker
      Specifies the string that marks a line which PMD should ignore; default
      is NOPMD.
      Default: NOPMD
    -threads, -t
      Sets the number of threads used by PMD.
      Default: 1
    -uri, -u
      Database URI for sources.
    -debug, -verbose, -D, -V
      Debug mode.
      Default: false
    -version, -v
      Specify version of a language PMD should use.
```