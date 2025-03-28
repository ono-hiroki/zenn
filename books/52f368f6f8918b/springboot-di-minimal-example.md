---
title: "Spring BootでDIを試す"
---

# Spring Bootで最小構成のDIを試す（CommandLineRunner編）

Spring Bootを使って、**最小限の構成で依存性注入（DI）を試す方法**を紹介します。  
今回は `CommandLineRunner` を使って、アプリケーション起動時に処理を自動実行するシンプルな例です。

---

## 🎯 ゴール

- クラス間で依存性注入を使う
- Spring Boot の起動時に処理を自動実行する
- 不要な抽象化（インターフェース）を省いたシンプル構成

---

## 📁 ディレクトリ構成

```plaintext
src/
└── main/
└── java/
└── com/
└── example/
└── learn_spring/
        ├── LearnSpringApplication.java
        ├── GreetingService.java
        ├── GreetingServiceImpl.java
        └── GreetingRunner.java
```

---

## 🧩 各ファイルの中身

### `LearnSpringApplication.java`

```java
package com.example.learn_spring;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class LearnSpringApplication {

	public static void main(String[] args) {
		SpringApplication.run(LearnSpringApplication.class, args);
	}

}
```

### `GreetingRunner.java`

```java
package com.example.learn_spring;

public interface GreetingService {
    void printGreeting();
}
```

### `GreetingServiceImpl.java`

```java
package com.example.learn_spring;

import org.springframework.stereotype.Service;

@Service
public class GreetingServiceImpl implements GreetingService {

    @Override
    public void printGreeting() {
        System.out.println("こんにちは、Spring！");
    }
}
```

### `GreetingRunner.java`

```java
package com.example.learn_spring;

import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class GreetingRunner implements CommandLineRunner {

    private final GreetingService greetingService;

    public GreetingRunner(GreetingService greetingService) {
        this.greetingService = greetingService;
    }

    @Override
    public void run(String... args) {
        greetingService.printGreeting();
    }
}
```


```bash
$ ./gradlew bootRun
```