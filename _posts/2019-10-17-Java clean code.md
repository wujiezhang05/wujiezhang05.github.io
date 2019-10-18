---
layout: default
title: Java clean code
category: Java
---

> 好代码--可以让人用更少的时间去理解ta。

---------------------------------------------
先来看一段代码，怎么来重构
```java
import java.util.ArrayList;
import java.util.Iterator;

public class ReportCard {

    private String studentName;
    private ArrayList<String> cLines;

    public void printReport(){
        System.out.println("Report card for " + studentName);
        System.out.println("----------------------");
        System.out.println("Course Title     Grade");

        Iterator grades = cLines.iterator();

        CourseGrade grade;
        double avg = 0.0;

        while(grades.hasNext()){
            grade = (CourseGrade) grades.next();
            System.out.println(grade.title + "    " + grade.grade);

            if (grade.grade == 'F'){
                avg = avg + grade.grade - 64;
            }
        }
        avg = avg/cLines.size();

        System.out.println("---------------------");
        System.out.println("Grade Point Average = " + avg);
    }
    class CourseGrade{
        String title;
        char grade;
    }
}
```

这段代码的主要问题在于
1. 违反单一抽象层次原则 （一个方法中所有操作最好处于同一个层次）：
 作为一个public方法，让人不能第一时间看出来意图。 正确的做法应该是
    printReportHeader（）
    printReportBody（）
    printReportFooter（）
所有操作都是一个抽象层次。
2. 变量的定义混乱：
变量 avg 在计算过程中 代表 总成绩，也代表平均成绩.
正确的做法是 再定义一个 sum 变量代表 总成绩。
3. 一个循环做了两件事情：
在一个循环内 既打印report的body，又计算总成绩。这时候我们会纠结
“ 明明可以一次循环弄好的事情，为什么要分成两个循环呢？ 这样性能不好吧。”
多数情况下 多一个循环对性能的影响是很小很小的（可以自己做实验验证）。 ------PS 如果要循环的数据结构很大 很花时间。那就保持一个循环。
4.  'F' 魔法数。 不知道具体含义:
  应该定义一个常量，并以容易理解的单词命名。让他人一眼知道其含义。
  
整理之后的代码为：

```java
import java.util.ArrayList;
import java.util.Iterator;

public class ReportCard {

    private String studentName;
    private ArrayList<String> cLines;
    private static char gradeLevel = 'F';

    public void printReport(){
        printHeader();

        printReportBody();

        double avg = computeAvg();

        printReportFooter(avg);
    }

    private void printHeader(){
        System.out.println("Report card for " + studentName);
        System.out.println("----------------------");
        System.out.println("Course Title     Grade");
    }

    private void printReportBody(){
        CourseGrade grade;
        Iterator grades = cLines.iterator();
        while(grades.hasNext()){
            grade = (CourseGrade) grades.next();
            System.out.println(grade.title + "    " + grade.grade);
        }
    }


    private void printReportFooter(double avg){
        System.out.println("---------------------");
        System.out.println("Grade Point Average = " + avg);
    }

    private double computeAvg(){
        Iterator grades = cLines.iterator();

        CourseGrade grade;
        double sum = 0.0;
        while(grades.hasNext()){
            grade = (CourseGrade) grades.next();
            if (grade.grade == gradeLevel){
                sum = sum + grade.grade - 64;
            }
        }
        return sum/cLines.size();
    }


    class CourseGrade{
        String title;
        char grade;
    }
}
``` 

-------------------------------------------

再来看下面的代码

```java
public String convert(String content, String operationType){
        if("001".equals(operationType)){
            return content.toLowerCase();
        }else if("002".equals(operationType)){
            return  content.toUpperCase();
        }else if("003".equals(operationType)){
            //do C
        }else if("004".equals(operationType)){
            //do D
        }else if("005".equals(operationType)){
            //do E
        }
    }
```
这段代码的问题是：
臃肿的if-else 结构， 每次增加新功能都要修改这一段代码体。理解起来也复杂。

怎么来重构呢？---- ***command 模式***
1. 把每块处理逻辑放到一个简单的“命令”类中，这个类有一个通用的方法。
2. 用集合来 存储/添加/删除 一批这些命令类的实例. 通过调用他们的执行方法来执行这些实例

上代码

```java
public interface MyConverter {
    public String convert(String content);
}

public class ToLowerCaseConverter implements MyConverter{
    public String convert(String content){
        return content.toLowerCase();
    }
}

public class ToUpperCaseConverter implements MyConverter {
    public String convert(String content){
        return content.toUpperCase();
    }
}

public class Dispatcher {

    private Map<String, MyConverter> handlers;

    public void createHandlers(){
        handlers = new HashMap<String, MyConverter>();
        handlers.put("001", new ToLowerCaseConverter());
        handlers.put("002", new ToUpperCaseConverter());
        //handlers.put("003", new .....);
        //handlers.put("004", new .....);
    }

    public String convert(String content, String operationType){
        MyConverter handler = lookupHandlerByType(operationType);
        return handler.convert(content);
    }

    private MyConverter lookupHandlerByType(String type){
        return handlers.get(type);
    }
}
public class Client{
    public static void main(String[] args){
        Dispatcher dispatcher = new Dispatcher();
        dispatcher.convert("ABC", "001");
    }
}
```

### 其他需要注意的点
1. 方法尽量不要传null，也不要返回null。 可以定义NullObject 来代替 null
2. 判断条件，if (condition) 用肯定句，尽量不要用 if（！condition），便于其他快速理解
3. 每一行 一个语句
4. 方法名 和 内容要一致。