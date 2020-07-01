git在commit之前，进行代码检查  
构建PMD工具包见[./ali-p3c.md]  
然后在项目根目录的`.git/hooks/`目录下面，可以看到很多的默认hook，不过都是以.sample结尾的，没有启用。  
如果要启用某个hook，则可以将文件重命名，去掉.sample后缀即可。  
此处我们可以自定义一个`pre-commit`文件，具体内容如下，只需要将`pec-pmd-*.jar`的jar包放在合适的目录即可。  
这样就会在commit之前进行代码检查，检查不通过则不能commit，当然也可以在提交的时候通过跳过该检查。
```
#!/bin/sh

echo "===== PMD begin ====="
REJECT=0

echo `pwd`

STAGE_FILES=$(git diff --cached --name-only --diff-filter=ACM -- '*.java' '*.xml')
if test ${#STAGE_FILES} -gt 0
then

  java -Dfile.encoding=utf-8 -cp .git/hooks/p3c-pmd-2.0.1-jar-with-dependencies.jar net.sourceforge.pmd.PMD -d ${STAGE_FILES} -cache pmd-cache -R rulesets/java/ali-comment.xml,rulesets/java/ali-concurrent.xml,rulesets/java/ali-constant.xml,rulesets/java/ali-exception.xml,rulesets/java/ali-flowcontrol.xml,rulesets/java/ali-naming.xml,rulesets/java/ali-oop.xml,rulesets/java/ali-orm.xml,rulesets/java/ali-other.xml,rulesets/java/ali-set.xml -f text
  REJECT=$?
  echo $REJECT

fi

if [ $REJECT == 0 ]
then 
	echo "===== PMD passed ====="
else
	echo "===== PMD 检查失败 ====="
fi

exit $REJECT
```