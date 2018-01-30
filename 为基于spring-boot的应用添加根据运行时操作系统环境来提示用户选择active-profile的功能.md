---
title: 为基于spring-boot的应用添加根据运行时操作系统环境来提示用户选择active profile的功能
date: 2018-01-12 15:07:58
tags:
---
`spring-boot`有一个根据`JVM`变量`-Dspring.profiles.active`来设置运行时的active profile的功能，但是有些时候我们也许会不小心忘记设置这个变量，这样在生产环境中会带来一定的困扰，所以我想了一个办法，来给忘记设置`-Dspring.profiles.active`的程序员一次“secend chance”。

{% asset_img 160e8d454dce61c0.png %}

先来讲一下思路：

- step0 约定好profiles的命名，“development”代表开发环境(也可以将默认的profile设为开发环境)，“production”代表生产环境
- step1 判断是否设置了`-Dspring.profiles.active`，如果已经设置，直接跳转`step3`
- step2 判断当前操作系统环境，如果不是Linux环境则认定为开发环境，自动倒计时激活开发的profile；如果是Linux环境则认定为生产环境，输出选择profile的控制台信息，并等待用户控制台输入进行选择，并依据用户选择来激活profile
- step3 `SpringApplication.run()`

代码如下：

spring-boot配置文件(使用了默认profile作为开发环境)：
```
spring:
  application:
    name: comchangyoueurekaserver #注意命名要符合RFC 2396，否则会影响服务发现 详见https://stackoverflow.com/questions/37062828/spring-cloud-brixton-rc2-eureka-feign-or-rest-template-configuration-not-wor

server:
  port: 8001

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}

---
spring:
  profiles: production
  application:
    name: comchangyoueurekaserver

server:
  port: 8001

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}
```
`BootStarter`封装了`step1`-`step3`的逻辑：

```
import org.apache.commons.lang3.StringUtils;

import java.util.Scanner;
import java.util.Timer;
import java.util.TimerTask;
import java.util.regex.Pattern;

public class BootStarter {

    //用于后续Spring Boot操作的回调
    public interface Callback {
        void bootRun();
    }

    private boolean enableAutomaticallyStart = true;
    private int automaticallyStartDelay = 10;

    public boolean isEnableAutomaticallyStart() {
        return enableAutomaticallyStart;
    }

    public void setEnableAutomaticallyStart(boolean enableAutomaticallyStart) {
        this.enableAutomaticallyStart = enableAutomaticallyStart;
    }

    public int getAutomaticallyStartDelay() {
        return automaticallyStartDelay;
    }

    public void setAutomaticallyStartDelay(int automaticallyStartDelay) {
        this.automaticallyStartDelay = automaticallyStartDelay;
    }

    public void startup(boolean enableAutomaticallyStart, int automaticallyStartDelay, Callback callback) {
        if (StringUtils.isBlank(System.getProperty("spring.profiles.active"))) { //如果没有通过参数spring.profiles.active设置active profile则让用户在控制台自己选择
            System.out.println("***Please choose active profile:***\n\tp: production\n\td: development");

            final boolean[] started = {false};
            Timer timer = new Timer();
            if (enableAutomaticallyStart && System.getProperty("os.name").lastIndexOf("Linux") == -1) { //如果当前操作系统环境为非Linux环境(一般为开发环境)则automaticallyStartDelay秒后自动设置为开发环境
                System.out.printf("\nSystem will automatically select 'd' in %d seconds.\n", automaticallyStartDelay);
                final int[] count = {automaticallyStartDelay};
                timer.scheduleAtFixedRate(new TimerTask() {
                    @Override
                    public void run() {
                        if (count[0]-- == 0) {
                            timer.cancel();
                            started[0] = true;

                            System.setProperty("spring.profiles.active", "development");
                            callback.bootRun();
                        }
                    }
                }, 0, 1000);
            }

            Scanner scanner = new Scanner(System.in);
            Pattern pattern = Pattern.compile("^p|d$");
            //如果是Linux系统(一般为生产环境)则强制等待用户输入(一般是忘记设置spring.profiles.active了，这等于给了设置active profile的"second chance")
            while (scanner.hasNextLine()) {
                if (started[0]) {
                    break;
                }
                String line = scanner.nextLine();
                if (!pattern.matcher(line).find()) {
                    System.out.println("INVALID INPUT!");
                } else {
                    timer.cancel();
                    System.setProperty("spring.profiles.active", line.equals("d") ? "development" : "production");
                    callback.bootRun();
                    break;
                }
            }
        } else { //如果已经通过参数spring.profiles.active设置了active profile直接启动
            callback.bootRun();
        }
    }

    public void startup(Callback callback) {
        startup(this.enableAutomaticallyStart, this.automaticallyStartDelay, callback);
    }
}

```

main()：

```
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
import org.springframework.context.ApplicationContext;

@EnableEurekaServer
@SpringBootApplication
public class App {

    private static final Logger LOGGER = LoggerFactory.getLogger(App.class);

    public static void main(String[] args) {
        new BootStarter().startup(() -> {
            ApplicationContext applicationContext = SpringApplication.run(App.class, args);
            for (String activeProfile : applicationContext.getEnvironment().getActiveProfiles()) {
                LOGGER.warn("***Running with profile: {}***", activeProfile);
            }
        });
    }
}
```

运行效果(开发环境Mac OS)：
{% asset_img 160e8d815d8e13fd.png %}

扩展：
其实在这里我们还可以发散一下思维，基于spring-boot的应用比起传统spring应用的一大优势是自己可以掌控`main()`方法，有了这一点，我们是能玩出很多花样来的，思路不要被局限在`tomcat`时代了。

**main法在手，天下我有。**

---
2017-1-22更新：增加了线程安全的处理
```
import org.apache.commons.lang3.StringUtils;

import java.util.Scanner;
import java.util.Timer;
import java.util.TimerTask;
import java.util.regex.Pattern;

public class BootStarter {

    private volatile boolean started;

    //用于后续Spring Boot操作的回调
    public interface Callback {
        void bootRun();
    }

    private boolean enableAutomaticallyStart = true;
    private int automaticallyStartDelay = 3;

    public boolean isEnableAutomaticallyStart() {
        return enableAutomaticallyStart;
    }

    public void setEnableAutomaticallyStart(boolean enableAutomaticallyStart) {
        this.enableAutomaticallyStart = enableAutomaticallyStart;
    }

    public int getAutomaticallyStartDelay() {
        return automaticallyStartDelay;
    }

    public void setAutomaticallyStartDelay(int automaticallyStartDelay) {
        this.automaticallyStartDelay = automaticallyStartDelay;
    }

    public void startup(boolean enableAutomaticallyStart, int automaticallyStartDelay, Callback callback) {
        if (StringUtils.isBlank(System.getProperty("spring.profiles.active"))) { //如果没有通过参数spring.profiles.active设置active profile则让用户在控制台自己选择
            System.out.println("***Please choose active profile:***\n\tp: production\n\td: development");

            Timer timer = new Timer();
            if (enableAutomaticallyStart && System.getProperty("os.name").lastIndexOf("Linux") == -1) { //如果当前操作系统环境为非Linux环境(一般为开发环境)则automaticallyStartDelay秒后自动设置为开发环境
                System.out.printf("\nSystem will automatically select 'd' in %d seconds.\n", automaticallyStartDelay);
                timer.scheduleAtFixedRate(new TimerTask() {

                    private ThreadLocal<Integer> countDown = ThreadLocal.withInitial(() -> automaticallyStartDelay);

                    @Override
                    public void run() {
                        if (countDown.get() == 0) {
                            timer.cancel();
                            started = true;

                            System.setProperty("spring.profiles.active", "development");
                            callback.bootRun();
                        }
                        countDown.set(countDown.get() - 1);
                    }
                }, 0, 1000);
            }

            Scanner scanner = new Scanner(System.in);
            Pattern pattern = Pattern.compile("^p|d$");
            //如果是Linux系统(一般为生产环境)则强制等待用户输入(一般是忘记设置spring.profiles.active了，这等于给了设置active profile的"second chance")
            while (scanner.hasNextLine()) {
                if (started) {
                    break;
                }
                String line = scanner.nextLine();
                if (!pattern.matcher(line).find()) {
                    System.out.println("INVALID INPUT!");
                } else {
                    timer.cancel();
                    System.setProperty("spring.profiles.active", line.equals("d") ? "development" : "production");
                    callback.bootRun();
                    break;
                }
            }
        } else { //如果已经通过参数spring.profiles.active设置了active profile直接启动
            callback.bootRun();
        }
    }

    public void startup(Callback callback) {
        startup(this.enableAutomaticallyStart, this.automaticallyStartDelay, callback);
    }
}

```

---
2017-01-25更新：

补上了 `try-with-resource`
```
import org.apache.commons.lang3.StringUtils;

import java.util.Scanner;
import java.util.Timer;
import java.util.TimerTask;
import java.util.regex.Pattern;

public class BootStarter {

    private volatile boolean started;

    //用于后续Spring Boot操作的回调
    public interface Callback {
        void bootRun();
    }

    private boolean enableAutomaticallyStart = true;
    private int automaticallyStartDelay = 3;

    public boolean isEnableAutomaticallyStart() {
        return enableAutomaticallyStart;
    }

    public void setEnableAutomaticallyStart(boolean enableAutomaticallyStart) {
        this.enableAutomaticallyStart = enableAutomaticallyStart;
    }

    public int getAutomaticallyStartDelay() {
        return automaticallyStartDelay;
    }

    public void setAutomaticallyStartDelay(int automaticallyStartDelay) {
        this.automaticallyStartDelay = automaticallyStartDelay;
    }

    public void startup(boolean enableAutomaticallyStart, int automaticallyStartDelay, Callback callback) {
        if (StringUtils.isBlank(System.getProperty("spring.profiles.active"))) { //如果没有通过参数spring.profiles.active设置active profile则让用户在控制台自己选择
            System.out.println("***Please choose active profile:***\n\tp: production\n\td: development");

            Timer timer = new Timer();
            if (enableAutomaticallyStart && System.getProperty("os.name").lastIndexOf("Linux") == -1) { //如果当前操作系统环境为非Linux环境(一般为开发环境)则automaticallyStartDelay秒后自动设置为开发环境
                System.out.printf("\nSystem will automatically select 'd' in %d seconds.\n", automaticallyStartDelay);
                timer.scheduleAtFixedRate(new TimerTask() {

                    private ThreadLocal<Integer> countDown = ThreadLocal.withInitial(() -> automaticallyStartDelay);

                    @Override
                    public void run() {
                        if (countDown.get() == 0) {
                            timer.cancel();
                            started = true;

                            System.setProperty("spring.profiles.active", "development");
                            callback.bootRun();
                        }
                        countDown.set(countDown.get() - 1);
                    }
                }, 0, 1000);
            }

            try (Scanner scanner = new Scanner(System.in)) {
                Pattern pattern = Pattern.compile("^p|d$");
                //如果是Linux系统(一般为生产环境)则强制等待用户输入(一般是忘记设置spring.profiles.active了，这等于给了设置active profile的"second chance")
                while (scanner.hasNextLine()) {
                    if (started) {
                        break;
                    }
                    String line = scanner.nextLine();
                    if (!pattern.matcher(line).find()) {
                        System.out.println("INVALID INPUT!");
                    } else {
                        timer.cancel();
                        System.setProperty("spring.profiles.active", line.equals("d") ? "development" : "production");
                        callback.bootRun();
                        break;
                    }
                }
            }
        } else { //如果已经通过参数spring.profiles.active设置了active profile直接启动
            callback.bootRun();
        }
    }

    public void startup(Callback callback) {
        startup(this.enableAutomaticallyStart, this.automaticallyStartDelay, callback);
    }
}

```