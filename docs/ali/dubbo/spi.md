# SPI 机制
## Java 所使用的的 SPI 

demo：

```java
public interface Robot {
    void sayHello();
}

public class ARobot implements Robot {
    @Override
    public void sayHello() {
        System.out.println("hello a robot");
    }
}

public class BRobot implements Robot {
    @Override
    public void sayHello() {
        System.out.println("hello b robot");
    }
}
```

​		在`resources`目录下新建`META-INF/services`目录，并新建`Robot`类名全路径文件(com.study.Robot)，在该文件内增加如下内容

```
com.study.ARobot
com.study.BRobot
```



测试

```java
public static void main(String[] args) {
    ServiceLoader<Robot> serviceLoader = ServiceLoader.load(Robot.class);
    serviceLoader.forEach(Robot::sayHello);
}
```

## Dubbo 中的 SPI 机制


