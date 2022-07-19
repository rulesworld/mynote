# Gradle

## build.gradle中直接限制gradle版本，不使用gradle.properties

```gradle
task wrapper(type: Wrapper) {
    gradleVersion = '6.8.3'
}

// try this
wrapper {
    gradleVersion = '6.8.3'
}
```

## gralde按属性引入，适合DDD，多环境

```gradle
def profile = System.getProperty("profile")

apply from: "profile-${profile}.gradle"

bootRun {
    args = ["--spring.profiles.active=" + profile]
}
```

## gradle7 maven--->maven-publish plugin

https://docs.gradle.org/current/userguide/publishing_maven.html

## sprngboot与gralde继承概要

https://docs.spring.io/spring-boot/docs/current/gradle-plugin/reference/htmlsingle/#introduction