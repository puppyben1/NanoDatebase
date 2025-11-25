# OJ

## 判题模块

### 工厂模式优化

首先是创建代码沙箱实例，有多种代码沙箱的类型，如果要改变调用，需要手动改变 new 的对象，不方便，可以定义一个代码沙箱的静态工厂，用户可以传入一个类型字符串，让工厂自动改变 new 出来的对象。

```java
package com.axin.OJ.judge;

import com.axin.OJ.judge.impl.ExampleCodeSandbox;
import com.axin.OJ.judge.impl.RemoteCodeSandbox;
import com.axin.OJ.judge.impl.ThirdPartyCodeSandbox;

/**
 * 代码沙箱静态工厂
 */
public class CodeSandboxFactory {

	/**
	 * 创建代码沙箱
	 * @param type 沙箱类型
	 * @return 代码沙箱实例
	 */
	private static CodeSandbox newInstance(String type) {
		switch (type) {
			case "remote":
				return new RemoteCodeSandbox();
			case "third_party":
				return new ThirdPartyCodeSandbox();
			default:
				return new ExampleCodeSandbox();
		}
	}
}
-------------------------------------------------------------------------------
// 获取代码沙箱实例
CodeSandbox codeSandbox = CodeSandboxFactory.newInstance(codeSandboxType);
```



### 代理模式优化

有了代码沙箱实例，现在想要增强代码沙箱的功能，写点新的代码，且不惊动原有代码，那就需要类似AOP增强方法，代理对象用来增强目标对象的功能，一般就是代理对象和目标对象继承同一个接口，代理类中定义接口对象用来接受传进来的目标对象实例，编写新的语句增强相应方法的功能

```java
package com.axin.OJ.judge;

import com.axin.OJ.judge.model.ExecuteCodeRequest;
import com.axin.OJ.judge.model.ExecuteCodeResponse;
import lombok.extern.slf4j.Slf4j;

/**
 * 代码沙箱代理类
 */
@Slf4j
public class CodeSandboxProxy implements CodeSandbox{

    /**
	 * 代码沙箱接口，用来接受传进来的代码沙箱实例
	 */
    private final CodeSandbox codeSandbox;

    /**
	 * 构造方法，用来接受传进来的代码沙箱实例
	 * @param codeSandbox 代码沙箱实例
	 */
    public CodeSandboxProxy(CodeSandbox codeSandbox) {
        this.codeSandbox = codeSandbox;
    }

    /**
	 * 执行代码
	 * @param request 执行代码请求
	 * @return 执行代码响应
	 */
    @Override
    public ExecuteCodeResponse executeCode(ExecuteCodeRequest request) {
        log.info("代码沙箱请求开始执行:{}", request);
        ExecuteCodeResponse executeCodeResponse = codeSandbox.executeCode(request);
        log.info("代码沙箱响应结果");
        //		log.info("代码沙箱执行完成:{}", executeCodeResponse);
        return executeCodeResponse;
    }
}
-----------------------------------------------------------------------------------------
// 获取代码沙箱实例
CodeSandbox codeSandbox = CodeSandboxFactory.newInstance(codeSandboxType);
// 创建代码沙箱代理
codeSandbox = new CodeSandboxProxy(codeSandbox);
// 调用代码沙箱执行代码 (执行状态 执行信息 JudgeInfo 输出用例)
ExecuteCodeResponse executeCodeResponse = codeSandbox.executeCode(executeCodeRequest);
```



### 策略模式优化

现在假设代码沙箱已经给我们返回了结果，那我是只准备一个判题策略呢，还是根据情况准备多个，这就是策略模式，写一个策略接口，实现几个策略类，再写一个管理类决定调用那种策略，一般是根据传给管理类的参数决定。目前是根据语言决定使用那个策略。

```java
public class JudgeManager {

	/**
	 * 选定判题策略
	 * @param judgeContext 判题上下文
	 * @return 判题信息
	 */
	public JudgeInfo doJudge(JudgeContext judgeContext) {
		// 获取判题记录
		QuestionSubmit questionSubmit = judgeContext.getQuestionSubmit();
		// 获取语言 实际上目前选择策略就是根据语言来选择策略
		String language = questionSubmit.getLanguage();
		JudgeStrategy judgeStrategy = new DefaultJudgeStrategy();
		if("java".equals(language)){
			judgeStrategy = new JavaJudgeStrategy();
		}
		return judgeStrategy.doJudge(judgeContext);
	}
}
---------------------------------------------------------------------------------------
```

```java
/**
 * 判题策略  是代码沙箱执行完成之后反馈的数据和题目的要求都有了才开始判题
 */
public interface JudgeStrategy {
	/**
	 * 执行判题
	 * @param judgeContext 判题上下文
	 * @return 判题信息
	 */
	JudgeInfo doJudge(JudgeContext judgeContext);
}
```

```java
/**
 * Java程序的判题策略
 */
public class JavaJudgeStrategy implements JudgeStrategy {
	/**
	 * 执行判题
	 * @param judgeContext 判题上下文
	 * @return 判题信息
	 */
	@Override
	public JudgeInfo doJudge(JudgeContext judgeContext) {
		// 要返回的结果
		JudgeInfo judgeInfoResult = new JudgeInfo();
		// 获取沙箱执行完返回的参数和题目要求参数
		// 获取执行代码的时间，内存和运行信息
		JudgeInfo judgeInfo = judgeContext.getJudgeInfo();
		// 从题目获取的输入用例
		List<String> inputList = judgeContext.getInputList();
		// 输出用例
		List<String> outputList = judgeContext.getOutputList();
		// 标准答案
		List<JudgeCase> judgeCaseList = judgeContext.getJudgeCaseList();
		// 原题
		Question question = judgeContext.getQuestion();
		// 先设置执行本身消耗的时间和内存
		judgeInfoResult.setMemoryConsumption(judgeInfo.getMemoryConsumption());
		judgeInfoResult.setTimeConsumption(judgeInfo.getTimeConsumption());
		// 根据代码沙箱执行的结果，设置本次提交的状态和信息 也就是runMessage
		JudgeInfoMessageEnum judgeInfoMessageEnum = JudgeInfoMessageEnum.WAITING;
		// 下面根据沙箱返回的结果和题目设置的标准对比设置judgeInfoMessageEnum
		// 首先根据输出用例个数和输入用例个数判断是否匹配
		if (outputList.size() != inputList.size()) {
			judgeInfoMessageEnum = JudgeInfoMessageEnum.WRONG_ANSWER;
			judgeInfoResult.setRunMessage(judgeInfoMessageEnum.getValue());
			return judgeInfoResult;
		}
		// 接着一个一个比对用例是否匹配
		for (int i = 0; i < judgeCaseList.size(); i++) {
			if (!outputList.get(i).equals(judgeCaseList.get(i).getOutput())) {
				judgeInfoMessageEnum = JudgeInfoMessageEnum.WRONG_ANSWER;
				judgeInfoResult.setRunMessage(judgeInfoMessageEnum.getValue());
				return judgeInfoResult;
			}
		}
		// 比对时间和内存消耗
		String judgeConfig = question.getJudgeConfig();
		if (judgeConfig == null || judgeConfig.isEmpty()) {
			throw new BusinessException(ErrorCode.PARAMS_ERROR, "题目判题配置为空");
		}
		// 把json字符串转换为JudgeConfig对象
		JudgeConfig judgeConfigObj = JSONUtil.toBean(judgeConfig, JudgeConfig.class);
		if (judgeConfigObj == null) {
			throw new BusinessException(ErrorCode.PARAMS_ERROR, "题目判题配置格式错误");
		}
		// 对比时间和内存消耗
		if (judgeContext.getJudgeInfo() != null) {
			// 时间消耗对比
			if (judgeContext.getJudgeInfo().getTimeConsumption() > judgeConfigObj.getTimeLimit()) {
				judgeInfoMessageEnum = JudgeInfoMessageEnum.TIME_LIMIT_EXCEEDED;
				judgeInfoResult.setRunMessage(judgeInfoMessageEnum.getValue());
				return judgeInfoResult;
			}
			// 内存消耗对比
			if (judgeContext.getJudgeInfo().getMemoryConsumption() > judgeConfigObj.getMemoryLimit()) {
				judgeInfoMessageEnum = JudgeInfoMessageEnum.MEMORY_LIMIT_EXCEEDED;
				judgeInfoResult.setRunMessage(judgeInfoMessageEnum.getValue());
				return judgeInfoResult;
			}
		}
		// 都没有问题，那么就是AC了
		judgeInfoMessageEnum = JudgeInfoMessageEnum.ACCEPT;
		judgeInfoResult.setRunMessage(judgeInfoMessageEnum.getValue());
		return judgeInfoResult;
	}
}

```

## 代码沙箱原生实现

总体思路：代码沙箱的作用就是你给我输入用例，我负责执行代码然后返回结果，所以本质就是 java 源文件 => Javac 编译 => Java 执行。这里采用新项目编写。

首先是参数校验，接着是在项目总目录下创建一个专门存放用户代码的目录，总目录下会有多个子目录，子目录名由 UUID 生成，子目录用于存放一个用户的代码，类名是Main（写死的），接着把代码写到 Main.java 文件中。

```java
/**
	 * 用来存放所有用户的代码
	 */
private static final String GLOBAL_CODE_DIR_NAME="tmpCode";
/**
	 * 总包下有多个UUID生成的子目录，每个子目录下有一个java文件，文件名固定为Main.java
	 */
private static final String GLOBAL_JAVA_CLASS_NAME="Main.java";

// 获取当前工作 目录
String currentPath = System.getProperty("user.dir");
// 用来存放所有代码的目录
String userCodePath = currentPath + File.separator + GLOBAL_CODE_DIR_NAME;
// 判断存放代码的目录是否存在，不存在则创建
if (!new File(userCodePath).exists()) {
    boolean mkdir = new File(userCodePath).mkdirs();
    if (!mkdir) {
        throw new RuntimeException("创建用户代码目录失败");
    }
}
// 将用户的代码写到对应的文件中，每一个用户有一个单独的目录，目录名是UUID
String userCodeParentPath = userCodePath + File.separator + UUID.randomUUID();
// UUID目录下的java文件路径
String userCodePathJava = userCodeParentPath + File.separator + GLOBAL_JAVA_CLASS_NAME;
// 将用户的代码写到java文件中 字符串写到文件中
File file = FileUtil.writeString(code, userCodePathJava, "utf-8");
```

下面执行编译命令，需要使用到 Process 类

```java
// 写进去之后就是一个java文件了，需要编译一下，准备命令
String compileCommand = String.format("javac -encoding utf-8 %s", userCodePathJava);
try {
    // 运行编译命令
    Process compileProcess = Runtime.getRuntime().exec(compileCommand);
    ExecuteMessage compileExecuteMessage = ProcessUtils.execute("编译", compileProcess);
    // 打印信息
    System.out.println(compileExecuteMessage);
} catch (IOException e) {
    throw new RuntimeException(e);
}
```

下面进入到 ProcessUtils 类中的 execute 方法，传进去操作名称、执行的进程，先开始计算时间消耗，然后获取进程退出的操作码（0是正常退出，其他都是有问题），如果是正常退出，获取进程的输入流（所谓输入输出是针对 Java 程序的），一行一行读取输出信息，设置到返回对象中，如果失败了，就获取正常输出信息和错误信息写到返回对象中，最后结算时间

```java
/**
 * 用来执行命令的工具类
 */
public class ProcessUtils {

	/**
	 * 执行命令
	 * @param operateName 操作名称
	 * @param process 命令进程
	 * @return 执行结果
	 */
	public static ExecuteMessage execute(String operateName, Process process) {
		// 要返回的对象
		ExecuteMessage executeMessage = new ExecuteMessage();
		try {
			StopWatch stopWatch = new StopWatch();
			stopWatch.start();
			// 获取进程结束的操作码
			int exitValue = process.waitFor();
			executeMessage.setExitValue(exitValue);
			// 正常退出
			if (exitValue == 0) {
				System.out.println(operateName + "成功");
				//分批获取进程的正常输出 所谓输入输出是针对JAVA程序的，程序获取就是从外部输入
				BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(process.getInputStream()));
				// 要返回的executeMessage属性--正常输出的信息
				StringBuilder message = new StringBuilder();
				String line;
				// 一行一行读取
				while ((line = bufferedReader.readLine()) != null) {
					message.append(line);
				}
				executeMessage.setMessage(message.toString());
				System.out.println(message);
			}
			else {
				//异常退出
				System.out.println(operateName + "失败，错误码" + exitValue);
				//分批获取进程的正常输出
				BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(process.getInputStream()));
				StringBuilder message = new StringBuilder();
				//逐行读取
				String compileOutputLine;
				while((compileOutputLine = bufferedReader.readLine())!=null){
					message.append(compileOutputLine);
				}
				executeMessage.setMessage(message.toString());
				//分批获取进程的错误输出
				BufferedReader errorBufferedReader = new BufferedReader(new InputStreamReader(process.getErrorStream()));
				StringBuilder errorMessage = new StringBuilder();
				//逐行读取
				String errorCompileOutputLine;
				while((errorCompileOutputLine = errorBufferedReader.readLine())!=null){
					errorMessage.append(errorCompileOutputLine);
				}
				executeMessage.setErrorMessage(errorMessage.toString());
			}
			// 计算运行时间
			stopWatch.stop();
			executeMessage.setTime(stopWatch.getTotalTimeMillis());
		} catch (InterruptedException | IOException e) {
			throw new RuntimeException(e);
		}
		return executeMessage;
	}
}
```

回到代码沙箱中，编译好之后，根据每一个输入用例依次执行 Java 命令

```java
// 这是每一个输入用例执行后的信息
List<ExecuteMessage> executeMessages = new ArrayList<>();
// 编译好Java文件，开始用输入用例一个一个运行
for (String input : inputList) {
    // 这里的Main是指类名，不是文件名，所以后面没有java的后缀
    String runCommand = String.format("java -Dfile.encoding=UTF-8 -cp %s Main %s", userCodeParentPath, input);
    try {
        // 运行命令
        Process runProcess = Runtime.getRuntime().exec(runCommand);
        ExecuteMessage runExecuteMessage = ProcessUtils.execute("运行", runProcess);
        // 运行结果
        System.out.println(runExecuteMessage);
        executeMessages.add(runExecuteMessage);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

封装好进程结束返回的信息集合，开始封装返回对象 ExecuteCodeResponse 

```java
public class ExecuteCodeResponse implements Serializable {

	private static final long serialVersionUID = 1L;

	/**
	 * 执行状态
	 */
	private Integer status;

	/**
	 * 执行结果信息 judgeInfo里面也有
	 */
	private String message;

	/**
	 * 时间消耗 内存消耗 程序执行信息
	 */
	private JudgeInfo judgeInfo;

	/**
	 * 输出用例
	 */
	private List<String> outputList;

}
```

首先看看这些输入实例的执行过程中有没有错误的，如果有，就直接设置错误信息，错误状态，如果没有就把所有的输出实例设置好，并得出所有执行进程的最大时间消耗（内存消耗太过于复杂，暂时不实现），然后封装 executeCodeResponse ，进行文件清理

```java
// 这是每一个输入用例执行后的信息
List<ExecuteMessage> executeMessages = new ArrayList<>();
// 编译好Java文件，开始用输入用例一个一个运行
for (String input : inputList) {
    // 这里的Main是指类名，不是文件名，所以后面没有java的后缀
    String runCommand = String.format("java -Dfile.encoding=UTF-8 -cp %s Main %s", userCodeParentPath, input);
    try {
        // 运行命令
        Process runProcess = Runtime.getRuntime().exec(runCommand);
        ExecuteMessage runExecuteMessage = ProcessUtils.execute("运行", runProcess);
        // 运行结果
        System.out.println(runExecuteMessage);
        executeMessages.add(runExecuteMessage);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
// 要返回的结果
ExecuteCodeResponse executeCodeResponse = new ExecuteCodeResponse();
// 输出用例集合
List<String> outputList = new ArrayList<>();
// 获取这些输入用例中运行时间最长的
long maxTime = 0;
// 看看有没有错误的
for (ExecuteMessage executeMessage : executeMessages) {
    if (executeMessage.getExitValue() != 0) {
        // 有错误的用例，记录错误信息
        executeCodeResponse.setMessage(executeMessage.getErrorMessage());
        // 设置错误状态
        executeCodeResponse.setStatus(3);
        break;
    }
    // 没错，就将输出信息加入到输出用例集合中，也就是结果3，结果7之类的
    outputList.add(executeMessage.getMessage());
    // 获取比较大的运行时间
    maxTime = Math.max(maxTime, executeMessage.getTime());
}
// 如果输出用例集合和每一个输入用例执行后的信息集合长度相等，说明都正确执行了
if (executeMessages.size() == outputList.size()) {
    // 设置正确执行状态
    executeCodeResponse.setStatus(1);
}
// 输出用例加到返回信息中
executeCodeResponse.setOutputList(outputList);
// 最大运行时间加到返回信息中
JudgeInfo judgeInfo = new JudgeInfo();
judgeInfo.setTimeConsumption(maxTime);
executeCodeResponse.setJudgeInfo(judgeInfo);
//5.文件清理
boolean del = FileUtil.del(userCodeParentPath);
System.out.println("文件清理" + (del ? "成功" : "失败"));
// 目前是只有错了才会给message属性赋值，如果是null，说明就是正确的
return executeCodeResponse;
```

到此，代码沙箱基本流程完成。

但是还是有一些安全问题。

### 无限睡眠（阻塞程序运行）

就是再提交的代码中让程序睡眠，占着进程不释放，时间上搞你

```java
package com.yupi.yuojcodesandbox.unsafe;

/**
 * 无限睡眠（阻塞程序执行）
 */
public class SleepError {
    public static void main(String[] args)throws InterruptedException {
        long ONE_HOUR=60*60*1000L;
        Thread.sleep(ONE_HOUR);
        System.out.println("睡晚了");
    }
}
```



### 无限创建资源

比如在一个死循环中无限的向集合中添加元素，空间上搞你，堆内存溢出，这是 JVM 的保护机制。

```java
package com.yupi.yuojcodesandbox.unsafe;

import java.util.ArrayList;
import java.util.List;

/**
 * 无限占用空间（浪费系统内存）
 */
public class MemoryError {
    public static void main(String[] args)throws InterruptedException {
        List<byte[]> bytes=new ArrayList<>();
        while(true){
            bytes.add(new byte[10000]);
        }
    }
}
```



可以使用 JVisualVM 和 JConsole 来观测 JVM 的状态

### 读文件，信息泄露

```java
package com.yupi.yuojcodesandbox.unsafe;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.List;

/**
 * 读取服务器文件（文件信息泄露）
 */
public class ReadFileError {
    public static void main(String[] args) throws InterruptedException, IOException {
        String userDir = System.getProperty("user.dir");
        String filePath=userDir+ File.separator+"/src/main/resources/application.yml";
        List<String> allLines = Files.readAllLines(Paths.get(filePath));
        System.out.println(String.join("\n",allLines));
    }
}
```

运行之后，你的配置文件的信息就会输出到控制台，被写到进程返回的 runExecuteMessage 中，进而被写到返回对象中给用户看，造成信息泄露

### 越权写文件

```java
package com.yupi.yuojcodesandbox.unsafe;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Arrays;
import java.util.List;

/**
 * 向服务器写文件（植入危险程序）
 */
public class WriteFileError {
    public static void main(String[] args) throws InterruptedException, IOException {
        String userDir = System.getProperty("user.dir");
        String filePath=userDir+ File.separator+"/src/main/resources/木马程序.bat";
        String errorProgram="java -version 2>&1";
        Files.write(Paths.get(filePath), Arrays.asList(errorProgram));
        System.out.println("写木马成功，你完了");
    }
}
```

根据路径用户可能会写一些危险的文件

### 执行危险程序

```java
package com.yupi.yuojcodesandbox.unsafe;

import java.io.BufferedReader;
import java.io.File;
import java.io.IOException;
import java.io.InputStreamReader;

/**
 * 运行其他程序（比如危险木马）
 */
public class RunFileError {
    public static void main(String[] args) throws InterruptedException, IOException {
        String userDir = System.getProperty("user.dir");
        String filePath=userDir+ File.separator+"src/main/resources/木马程序.bat";
        Process process = Runtime.getRuntime().exec(filePath);
        process.waitFor();
        //分批获取进程的正常输出
        BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(process.getInputStream()));
        //逐行读取
        String compileOutputLine;
        while((compileOutputLine = bufferedReader.readLine()) != null){
            System.out.println(compileOutputLine);
        }
        System.out.println("执行异常程序成功");
    }
}
```

能写就能执行

## 安全处理策略

### 超时控制

可以开启一个异步的守护线程，用来监控主线程的执行，例如守护线程睡了五秒，如果五秒守护线程睡醒了主线程还没执行完，我就直接杀死主线程

```java
// 编译好Java文件，开始用输入用例一个一个运行
for (String input : inputList) {
    // 这里的Main是指类名，不是文件名，所以后面没有java的后缀
    String runCommand = String.format("java -Dfile.encoding=UTF-8 -cp %s Main %s", userCodeParentPath, input);
    try {
        // 运行命令
        Process runProcess = Runtime.getRuntime().exec(runCommand);
        // 超时控制
        new Thread(() -> {
            try {
                // 先让这个线程睡几秒，如果用户执行完了还没有睡醒，那醒了之后JVM会自动回收线程
                Thread.sleep(TIME_OUT);
                System.out.println("超时了，中断");
                // 如果睡醒了执行命令的线程还没有结束，直接杀死
                if (runProcess.isAlive()) {
                    runProcess.destroy();
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }).start();
        ExecuteMessage runExecuteMessage = ProcessUtils.execute("运行", runProcess);
        // 运行结果
        System.out.println(runExecuteMessage);
        executeMessages.add(runExecuteMessage);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

如果说执行正常的话，不超时，开启的异步线程会被JVM自行回收

### 资源控制

执行 Java 命令的时候指定线程的资源  

```java
java -Xmx256m
----------------------
java -Xms256m
```

> 这个限制的并不是电脑系统总的资源，只是 JVM 堆的资源，所以实际占用的电脑资源肯定比你设置的值更高
>
> 如果说想要更加严格的限制，就要在系统层面去做限制，使用系统的命令去限制进程资源的分配

小知识-常用 JM 启动参数

1.内存相关参数:

O -Xms: 设置 JM 的初始堆内存大小

O -Xmx: 设置 JM 的最大堆内存大小

O -Xss: 设置线程的栈大小。

O -XX:MaxMetaspacesize: 设置 Metaspace(元空间)的最大大小。

O -XX:MaxDirectMemorySize:设置直接内存(Direct Memory)的最大大小

2.垃圾回收相关参数:

O -XX:+UseserialGc: 使用串行垃圾回收器。

O -XX:+UseParallelGc: 使用并行垃圾回收器。

O -XX:+UseConcMarksweepGC: 使用 CMS 垃圾回收器

O -XX:+UseG1GC: 使用 G1 垃圾回收器

3.线程相关参数:

O -XX:ParallelGCThreads: 设置并行垃圾回收的线程数,

O -XX:ConcGCThreads: 设置并发垃圾回收的线程数。

O -XX:ThreadStackSize: 设置线程的栈大小。

### 限制代码——黑白名单

我们可以将一些可能威胁到程序的单词放到字典树中对用户的代码进行校验

```java
/**
	 * 敏感词列表
	 */
private static final List<String> BAN_WORDS = Arrays.asList("File","System", "exec", "Runtime", "Process", "ProcessBuilder", "Thread", "File", "FileOutputStream", "FileInputStream", "FileWriter", "FileReader", "FileNotFoundException", "IOException", "BufferedReader", "BufferedWriter", "Scanner", "PrintWriter", "StringTokenizer", "StringBuilder", "StringBuffer", "String", "Integer", "Long", "Double", "Float", "Math");

/**
	 * 敏感词树,字典树
	 */
private static final WordTree BAN_WORD_TREE;
static {
    BAN_WORD_TREE = new WordTree();
    for (String banWord : BAN_WORDS) {
        BAN_WORD_TREE.addWord(banWord);
    }
}
```



```java
//校验代码
if (BAN_WORD_TREE.isMatch(code)) {
    throw new IllegalArgumentException("代码中有敏感词");
}
```



缺点就是你无法遍历所有的关键词，谁写什么你无法判断，再者就是每一个语言和领域涉及到的专有名词都不一样，人工限制的成本太大

#### 字典树

- 

### 限制权限

使用 Java 本身的安全管理器 SecurityManager 来严格的限制用户的操作权限，什么能读什么不能读，继承这个类就行，重写不同的方法对相应的操作进行限制

```java
/**
 * 默认的代码安全管理器,放开所有权限
 */
public class DefaultSecurityManager extends SecurityManager{

	/**
	 * 默认的代码安全管理器,放开所有权限
	 * @param perm 权限
	 */
	@Override
	public void checkPermission(Permission perm) {
		System.out.println("默认的代码安全管理器,放开所有权限");
		System.out.println(perm);
	}
}

```

```java
/**
 * 默认的代码安全管理器,拒绝所有权限
 */
public class DenySecurityManager extends SecurityManager{

	/**
	 * 默认的代码安全管理器,拒绝所有权限
	 * @param perm 权限
	 */
	@Override
	public void checkPermission(Permission perm) {
		System.out.println("默认的代码安全管理器,拒绝所有权限");
		// 所谓的拒绝就是直接抛异常
		throw new SecurityException("拒绝所有权限");
	}
}
```

可以根据自己的操作来进行相应的权限控制

```java
/**
 * 我的安全管理器，注意并不是对你的操作做什么调整，是直接拦截抛异常
 */
public class MySecurityManager extends SecurityManager{

	//放行所有的权限
	@Override
	public void checkPermission(Permission perm) {
        super.checkPermission(perm);
	}
	//检测程序是否可以执行外部命令，目前是拦截所有命令
	@Override
	public void checkExec(String cmd) {
		throw new SecurityException("checkExec 权限异常:"+cmd);
	}
	//检测程序是否可读文件
	@Override
	public void checkRead(String file) {
		System.out.println(file);
		// 这个目录下的文件可以读取，读取其他文件直接拦截
		if(file.contains("E:\\xiang-mu-he-ji\\OJ\\yuoj-code-sandbox")){
			return;
		}
        throw new SecurityException("checkRead 权限异常:"+file);
	}
	//检测程序是否写文件，目前是只要写就拦截
	@Override
	public void checkWrite(String file) {
		throw new SecurityException("checkWrite 权限异常:"+file);
	}
	//检测程序是否允许删除文件，目前是只要删除就拦截
	@Override
	public void checkDelete(String file) {
		throw new SecurityException("checkDelete 权限异常:"+file);
	}
	//检测程序是否允许连接网络，目前是只要连接就拦截
	@Override
	public void checkConnect(String host, int port) {
		throw new SecurityException("checkConnect 权限异常:"+host+":"+port);
	}
}
```

但是一般不会用 SecurityManager ，JDK17 都已经移除了这块代码，后面会采用容器化和虚拟机的方式执行。限制的时候不要限制在外层开发者写的代码，在执行用户程序的命令的时候开启 SecurityManager 。

```java
String runCommand = String.format("java -Xmx256m -Dfile.encoding=UTF-8 -Djava.security.manager=security.MySecurityManager -cp %s Main %s", userCodeParentPath, input);

```

安全管理器存在的问题

- 粒度太细，如果需要精细化的控制，你得自己一个一个想哪些权限需要限制
- 限制本身也是 Java 代码，会存在漏洞，而且只是程序层面的限制，并不是系统层面的限制

优点

- 太灵活了，而且实现方便

## Docker 实现代码沙箱

在虚拟机上安装 docker 之后，我们要使用 Java 操作 docker ，安装 docker - java 依赖，使用客户端操作。

在此之前，需要实现远程开发，我们的代码沙箱程序要运行在安装了 docker 的 Linux 上，才能使用 docker 来实现代码沙箱。先把代码同步到 Ubuntu 上，然后进行远程开发。

**Docker 代码沙箱**的实现思路

1. 直到编译之前的操作都不变
2. 编译好之后开始获取 docker 客户端，拉取 openjdk:8-alpine 的镜像，是轻量级的运行 Java 代码的环境，同时定义一个开关，Ubuntu 的 docker 中只有一个 Java 8 的镜像。

```java
// 编译完成，获取docker客户端
		DockerClient dockerClient = DockerClientBuilder.getInstance().build();
		// 拉取Java8的镜像
		String image = "openjdk:8-alpine";
		// 设置一个开关，如果是第一次拉去镜像就拉，拉过了就不用拉了
		if (FIRST_PULL) {
			// 创建拉取镜像的命令
			PullImageCmd pullImageCmd=dockerClient.pullImageCmd(image);
			// 拉取过程中的回调函数，每下载一个阶段调用一次
			PullImageResultCallback pullImageResultCallback=new PullImageResultCallback(){
				@Override
				public void onNext(PullResponseItem item) {
					System.out.println("下载镜像"+item.getStatus());
					super.onNext(item);
				}
			};
			try {
				// 拉取成功，开关关闭
				FIRST_PULL = false;
				// 拉取镜像是异步执行的，要等它执行完
				pullImageCmd.exec(pullImageResultCallback).awaitCompletion();
			} catch (InterruptedException e) {
				System.out.println("拉取镜像异常");
				// 如果拉取失败，开关依旧打开
				FIRST_PULL = true;
				throw new RuntimeException(e);
			}
		}
		System.out.println("下载完成");
```

3. 镜像下载完成之后，开始创建镜像的容器，并给这个容器进行一些配置

```java
// 下面开始创建容器，指定一下哪个镜像
		CreateContainerCmd containerCmd = dockerClient.createContainerCmd(image);
		// 给创建出来的容器一些配置
		HostConfig hostConfig=new HostConfig();
		// 内存限制
		hostConfig.withMemory(100*1000*1000L);
		// CPU限制
		hostConfig.withCpuCount(1L);
		// 将存储用户代码的目录挂载到容器的/app目录下
		hostConfig.setBinds(new Bind(userCodeParentPath,new Volume("/app")));
		// 创建容器
		CreateContainerResponse createContainerResponse = containerCmd
				.withHostConfig(hostConfig)
				.withNetworkDisabled(true)
				.withReadonlyRootfs(true)
				.withAttachStdin(true)  //输入流
				.withAttachStderr(true)  //错误流
				.withAttachStdout(true)  //输出流
				.withTty(true)  //支持交互模式
				.exec();
		System.out.println(createContainerResponse);
```

4. 容器创建好之后还要启动。然后就是循环输入用例列表，依次执行用户的代码程序。首先是创建在环境里执行字节码文件的命令（就是 Java 命令），同时还要指定在哪个容器内执行这个命令，这样命令 id 就和容器 id 绑定了，并配置命令的输入输出错误流

```java
// 用来存储每个输入用例执行后的结果信息
ArrayList<ExecuteMessage> executeMessages = new ArrayList<>();
// 一个一个输入执行
for (String inputArgs : inputList) {
    // 开启计时器
    StopWatch stopWatch = new StopWatch();
    // 将“1 2”空格分割数组
    String[] inputArgsArray = inputArgs.split(" ");
    // 拼接命令，每一个输入用例都要创建一次
    String[] cmdArray = ArrayUtil.append(new String[]{"java", "-cp", "/app", "Main"}, inputArgsArray);
    // 创建命令，指定一下容器的id，这里命令id和容器id也就绑定了
    ExecCreateCmdResponse execCreateCmdResponse = dockerClient.execCreateCmd(containerId)
        .withCmd(cmdArray) // 命令数组
        .withAttachStderr(true)  // 错误流
        .withAttachStdin(true)  //输入流
        .withAttachStdout(true) // 输出流
        .exec();
    System.out.println("创建执行命令：" + execCreateCmdResponse);
    ...........
}
```

5. 接着指定一下执行命令过程中的输入输出回调函数，用来获取输入输出信息

```java
// 此次执行的返回对象
ExecuteMessage executeMessage = new ExecuteMessage();
// 存储输出和错误信息（用数组是因为匿名类中需访问外部变量，数组是引用类型可修改）
// 输出信息
final String[] message = {null};
// 错误信息
final String[] errorMessage = {null};
// 程序执行时间
long time = 0L;
// 获取创建好的命令id，用于后续执行
String execId = execCreateCmdResponse.getId();
// 创建执行命令，用于收集输出和错误信息，收集信息回调函数
ExecStartResultCallback execStartResultCallback = new ExecStartResultCallback(){
    @Override
    public void onNext(Frame frame) {
        // 获取流用于判断是错误还是正常
        StreamType streamType = frame.getStreamType();
        // 错了
        if(StreamType.STDERR.equals(streamType)){
            errorMessage[0] =new String(frame.getPayload());
            System.out.println("输出错误结果："+ errorMessage[0]);
        }else{
            // 正常
            message[0] =new String(frame.getPayload());
            System.out.println("输出结果："+ message[0]);
        }
        super.onNext(frame);
    }
};
```

6. 然后配置命令执行过程中的资源管理器，用来统计内存等资源占用情况，内存占用是时刻发生变化的，所以我们就是定义一个周期获取内存

```java
// 存储内存
final long[] maxMemory = {0L};
// 指定一个容器执行代码时，要获取内存等资源统计信息
StatsCmd statsCmd = dockerClient.statsCmd(containerId);
// 内存获取回调函数
ResultCallback<Statistics> statisticsResultCallback = statsCmd.exec(new ResultCallback<Statistics>() {
    // 内存时刻发生着变化，我们只能定一个周期看，定期的获取内存
    @Override
    public void onNext(Statistics statistics) {
        System.out.println("内存占用：" + statistics.getMemoryStats().getUsage());
        maxMemory[0] =Math.max(statistics.getMemoryStats().getUsage(), maxMemory[0]);
    }
    @Override
    public void onStart(Closeable closeable) {

    }
    @Override
    public void onError(Throwable throwable) {

    }
    @Override
    public void onComplete() {

    }
    @Override
    public void close() throws IOException {

    }
});
// 设置获取内存等资源信息的回调函数
statsCmd.exec(statisticsResultCallback);
```

7. 然后开始在这个容器的环境里面执行用户的代码。这里的超时控制策略和下面有点不一样，这里是根据 awaitCompletion 的返回值来判断是否超时，然后封装消息，包括退出值、输出信息、错误信息、时间消耗、内存消耗

```java
// 开始执行用户程序
try {
    // 开始计时
    stopWatch.start();
    // 开始操作docker执行命令，前面有创建出来的命令的id，和获取信息的回调函数，因为是异步执行命令所有要等待执行完
    // awaitCompletion返回boolean：true表示正常完成，false表示超时
    boolean completed = dockerClient.execStartCmd(execId).exec(execStartResultCallback).
        awaitCompletion(TIME_OUT, TimeUnit.MILLISECONDS);
    // 停止计时
    stopWatch.stop();
    // 获取运行时间
    time= stopWatch.getLastTaskTimeMillis();
    // 关闭获取内存等资源信息的回调函数
    statsCmd.close();
    // 判断是否超时
    if (!completed) {
        // 超时了，设置exitValue为-1表示超时
        executeMessage.setExitValue(ExecuteMessageConstant.TIME_OUT);
        executeMessage.setErrorMessage("程序执行超时");
        System.out.println("程序执行超时");
    } else {
        // 正常完成，设置exitValue为0
        executeMessage.setExitValue(ExecuteMessageConstant.SUCCESS);
    }
} catch (InterruptedException e) {
    System.out.println("程序执行中断");
    throw new RuntimeException(e);
}
// 封装信息
executeMessage.setMessage(message[0]);
// 如果不是超时，才设置回调中的错误信息（超时的错误信息已经设置过了）
if (executeMessage.getExitValue() != ExecuteMessageConstant.TIME_OUT) {
    executeMessage.setErrorMessage(errorMessage[0]);
}
executeMessage.setTime(time);
executeMessage.setMemory(maxMemory[0]);
executeMessages.add(executeMessage);
```

8. 下面就是封装 executeCodeResponse 了，和 Java 原生代码沙箱一模一样。 

## Docker 代码沙箱的安全性问题

### 超时判断 

```java
dockerClient.execStartCmd(execId).exec(execStartResultCallback).
    awaitCompletion(TIME_OUT, TimeUnit.SECONDS);
```

执行命令的时候设置一个超时时间，但是，即使超时了，方法也不会抛出异常，而是直接停掉这个命令往下走，所以你不知道这个命令执行是否超时，因此，可以在获取信息的回调函数中设置一个 onComplete 方法，超时时间内执行完成了会调用，定义一个是否超时标志，默认超时，如果调用了方法就设置为不超时，来指明这次命令执行是否超时

### 内存资源

设置 HostConfig 的时候可以指定内存限制

```java
// 下面开始创建容器，指定一下哪个镜像
CreateContainerCmd containerCmd = dockerClient.createContainerCmd(image);
// 给创建出来的容器一些配置
HostConfig hostConfig=new HostConfig();
// 内存限制
hostConfig.withMemory(100*1000*1000L);
// CPU限制
hostConfig.withCpuCount(1L);
// 将存储用户代码的目录挂载到容器的/app目录下
hostConfig.setBinds(new Bind(userCodeParentPath,new Volume("/app")));
// 创建容器
CreateContainerResponse createContainerResponse = containerCmd
    .withHostConfig(hostConfig)
    .withAttachStdin(true)  //输入流
    .withAttachStderr(true)  //错误流
    .withAttachStdout(true)  //输出流
    .withTty(true)  //支持交互模式
    .exec();
```



### 网络资源限制

```java
// 创建容器
CreateContainerResponse createContainerResponse = containerCmd
    .withHostConfig(hostConfig)
    .withNetworkDisabled(true)  //限制访问网络
    .withAttachStdin(true)  //输入流
    .withAttachStderr(true)  //错误流
    .withAttachStdout(true)  //输出流
    .withTty(true)  //支持交互模式
    .exec();
```

### 权限控制

```java
// 创建容器
CreateContainerResponse createContainerResponse = containerCmd
    .withHostConfig(hostConfig)
    .withNetworkDisabled(true)
    .withReadonlyRootfs(true)
    .withAttachStdin(true)  //输入流
    .withAttachStderr(true)  //错误流
    .withAttachStdout(true)  //输出流
    .withTty(true)  //支持交互模式
    .exec();
```



比如这里根目录只能读。

Docker 容器已经做了系统层面的隔离，相对安全了，但是最好可以结合 Java 安全管理器一起使用更好。

## 模板方法优化代码沙箱

将代码中的步骤进行抽象，每一块需要做什么都抽出来一个方法，形成一套完整的流程

```java
package com.axin.codesandbox.codesandbox;

import cn.hutool.core.io.FileUtil;
import cn.hutool.dfa.WordTree;
import com.axin.codesandbox.model.ExecuteCodeRequest;
import com.axin.codesandbox.model.ExecuteCodeResponse;
import com.axin.codesandbox.model.ExecuteMessage;
import com.axin.codesandbox.model.JudgeInfo;
import com.axin.codesandbox.utils.ProcessUtils;

import java.io.File;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.UUID;

/**
 * 代码沙箱模板类
 */
public class JavaCodeSandboxTemplate implements CodeSandbox {

	/**
	 * 用来存放所有用户的代码
	 */
	private static final String GLOBAL_CODE_DIR_NAME="tmpCode";

	/**
	 * 总包下有多个UUID生成的子目录，每个子目录下有一个java文件，文件名固定为Main.java
	 */
	private static final String GLOBAL_JAVA_CLASS_NAME="Main.java";

	/**
	 * 敏感词列表
	 */
	private static final List<String> BAN_WORDS = Arrays.asList("File", "exec");

	/**
	 * 敏感词树,字典树
	 */
	private static final WordTree BAN_WORD_TREE;
	static {
		BAN_WORD_TREE = new WordTree();
		for (String banWord : BAN_WORDS) {
			BAN_WORD_TREE.addWord(banWord);
		}
	}

	/**
	 * 超时时间，单位毫秒
	 */
	public static final long TIME_OUT = 5000;

	/**
	 * 执行代码
	 *
	 * @param request 执行代码请求
	 * @return 执行代码响应
	 */
	@Override
	public ExecuteCodeResponse executeCode(ExecuteCodeRequest request) {
		// 检查参数
		checkExecuteCodeRequest(request);

		// 将用户的代码写到文件中
		File userCodeFile = saveCodeToFile(request.getCode());

		// 编译文件
		ExecuteMessage compileExecuteMessage = compileCode(userCodeFile);
		System.out.println(compileExecuteMessage);

		// 运行文件
		List<ExecuteMessage> executeMessages = runCode(userCodeFile, request.getInputList());

		// 处理运行结果
		ExecuteCodeResponse executeCodeResponse = handleRunResult(executeMessages);

		// 文件清理
		boolean deleteFile = deleteFile(userCodeFile);
		if (!deleteFile) {
			return getErrorResponse(new RuntimeException("文件清理失败"));
		}

		// 目前是只有错了才会给message属性赋值，如果是null，说明就是正确的
		return executeCodeResponse;
	}

	/**
	 * 保存代码到文件中
	 * @param code 用户输入的代码
	 * @return  文件
	 */
	public File saveCodeToFile(String code) {
		// 获取当前工作 目录
		String currentPath = System.getProperty("user.dir");
		// 用来存放所有代码的目录
		String userCodePath = currentPath + File.separator + GLOBAL_CODE_DIR_NAME;
		// 判断存放代码的目录是否存在，不存在则创建
		if (!new File(userCodePath).exists()) {
			boolean mkdir = new File(userCodePath).mkdirs();
			if (!mkdir) {
				throw new RuntimeException("创建用户代码目录失败");
			}
		}
		// 将用户的代码写到对应的文件中，每一个用户有一个单独的目录，目录名是UUID
		String userCodeParentPath = userCodePath + File.separator + UUID.randomUUID();
		// UUID目录下的java文件路径
		String userCodePathJava = userCodeParentPath + File.separator + GLOBAL_JAVA_CLASS_NAME;
		// 将用户的代码写到java文件中 字符串写到文件中
		return FileUtil.writeString(code, userCodePathJava, "utf-8");
	}

	/**
	 * 检查参数
	 * @param request 执行代码请求
	 */
	public void checkExecuteCodeRequest(ExecuteCodeRequest request) {
		// 参数校验
		if (request == null) {
			throw new IllegalArgumentException("执行代码请求不能为空");
		}
		if (request.getLanguage() == null || !request.getLanguage().equals("java")) {
			throw new IllegalArgumentException("只支持java语言");
		}
		if (request.getCode() == null) {
			throw new IllegalArgumentException("用户输入的代码不能为空");
		}
		if (request.getInputList() == null) {
			throw new IllegalArgumentException("输入的测试用例不能为空");
		}
		// 一组输入实例 目前形式为 “123 456”
		List<String> inputList = request.getInputList();
		// 用户输入的代码
		String code = request.getCode();
		//校验代码
		if (BAN_WORD_TREE.isMatch(code)) {
			throw new IllegalArgumentException("代码中有敏感词");
		}
	}

	/**
	 * 编译代码
	 * @param userCodeFile 用户代码文件
	 * @return 编译执行信息
	 */
	public ExecuteMessage compileCode(File userCodeFile) {
		// 写进去之后就是一个java文件了，需要编译一下，准备命令
		String compileCommand = String.format("javac -encoding utf-8 %s", userCodeFile.getAbsolutePath());
		try {
			// 运行编译命令
			Process compileProcess = Runtime.getRuntime().exec(compileCommand);
			ExecuteMessage compileExecuteMessage = ProcessUtils.execute("编译", compileProcess);
			// 打印信息
			System.out.println(compileExecuteMessage);
			return compileExecuteMessage;
		} catch (IOException e) {
			throw new RuntimeException(e);
		}
	}

	/**
	 * 运行代码
	 *
	 * @param userCodeFile 用户代码文件
	 * @param inputList    输入用例列表
	 * @return 输出用例列表
	 */
	public List<ExecuteMessage> runCode(File userCodeFile, List<String> inputList) {
		// 这是每一个输入用例执行后的信息
		List<ExecuteMessage> executeMessages = new ArrayList<>();
		// 编译好Java文件，开始用输入用例一个一个运行
		for (String input : inputList) {
			// 这里的Main是指类名，不是文件名，所以后面没有java的后缀
			String runCommand = String.format("java -Xmx256m -Dfile.encoding=UTF-8 -cp %s Main %s", userCodeFile.getParent(), input);
			try {
				// 运行命令
				Process runProcess = Runtime.getRuntime().exec(runCommand);
				// 超时控制
				new Thread(() -> {
					try {
						// 先让这个线程睡几秒，如果用户执行完了还没有睡醒，那醒了之后JVM会自动回收线程
						Thread.sleep(TIME_OUT);
						// 如果睡醒了执行命令的线程还没有结束，直接杀死
						if (runProcess.isAlive()) {
							System.out.println("超时了，中断");
							runProcess.destroy();
						}
					} catch (InterruptedException e) {
						throw new RuntimeException(e);
					}
				}).start();
				ExecuteMessage runExecuteMessage = ProcessUtils.execute("运行", runProcess);
				// 运行结果
				System.out.println(runExecuteMessage);
				executeMessages.add(runExecuteMessage);
			} catch (IOException e) {
				throw new RuntimeException(e);
			}
		}
		return executeMessages;
	}

	/**
	 * 处理运行结果
	 * @param executeMessages 运行结果
	 * @return 处理后的结果
	 */
	public ExecuteCodeResponse handleRunResult(List<ExecuteMessage> executeMessages) {
		// 要返回的结果
		ExecuteCodeResponse executeCodeResponse = new ExecuteCodeResponse();
		// 输出用例集合
		List<String> outputList = new ArrayList<>();
		// 获取这些输入用例中运行时间最长的
		long maxTime = 0;
		// 看看有没有错误的
		for (ExecuteMessage executeMessage : executeMessages) {
			if (executeMessage.getExitValue() != 0) {
				// 有错误的用例，记录错误信息
				executeCodeResponse.setMessage(executeMessage.getErrorMessage());
				// 设置错误状态
				executeCodeResponse.setStatus(3);
				break;
			}
			// 没错，就将输出信息加入到输出用例集合中，也就是结果3，结果7之类的
			outputList.add(executeMessage.getMessage());
			// 获取比较大的运行时间
			maxTime = Math.max(maxTime, executeMessage.getTime());
		}
		// 如果输出用例集合和每一个输入用例执行后的信息集合长度相等，说明都正确执行了
		if (executeMessages.size() == outputList.size()) {
			// 设置正确执行状态
			executeCodeResponse.setStatus(1);
		}
		// 输出用例加到返回信息中
		executeCodeResponse.setOutputList(outputList);
		// 最大运行时间加到返回信息中
		JudgeInfo judgeInfo = new JudgeInfo();
		judgeInfo.setTimeConsumption(maxTime);
		executeCodeResponse.setJudgeInfo(judgeInfo);
		return executeCodeResponse;
	}

	/**
	 * 文件清理
	 * @param userCodeFile 用户代码文件
	 * @return 是否删除成功
	 */
	public boolean deleteFile(File userCodeFile) {
		//5.文件清理
		boolean del = FileUtil.del(userCodeFile.getParent());
		System.out.println("文件清理" + (del ? "成功" : "失败"));
		return del;
	}

	/**
	 * 获取错误结果
	 * @param e 异常信息
	 * @return 错误结果
	 */
	private ExecuteCodeResponse getErrorResponse(Throwable e){
		ExecuteCodeResponse executeCodeResponse=new ExecuteCodeResponse();
		executeCodeResponse.setOutputList(new ArrayList<>());
		executeCodeResponse.setMessage(e.getMessage());
		//表示代码沙箱错误（可能是编译错误）
		executeCodeResponse.setStatus(2);
		executeCodeResponse.setJudgeInfo(new JudgeInfo());
		return executeCodeResponse;
	}
}
```

如果有其他执行代码的流程，只需要根据这个模板，复用并重写方法替换其中步骤即可，本质上是一个思想。

## 代码沙箱开放 API 

其实就是写一个 Controller 接口，用户传入请求参数就行。

```java
@Slf4j
@RestController
@RequestMapping("/code/sandbox")
public class CodeSandboxController {

	/**
	 * 执行代码
	 * @param executeCodeRequest 执行代码请求
	 * @return 执行代码响应
	 */
	@PostMapping("/execute")
	public ExecuteCodeResponse executeCode(@RequestBody ExecuteCodeRequest executeCodeRequest) {
		log.info("代码沙箱执行代码开始");
		log.info("代码沙箱执行代码请求参数：{}",executeCodeRequest);
		JavaNativeCodeSandbox javaNativeCodeSandbox = new JavaNativeCodeSandbox();
		return javaNativeCodeSandbox.executeCode(executeCodeRequest);
	}
}
```



### 调用安全性

目前项目还只是内部请求，相对可信，所以我们在服务之间发请求的时候，可以带上一个请求头，发起类和接受类都定义一下，接受的时候可以进行校验，如果不行就返回 403 。

```java
//内部调用时，用来鉴权的密钥
	private static final String HEADER_API_KEY = "request-key";

	private static final String SECRET_KEY = "secretKey";
```



