---
title:  "Spring batch step flow"

excerpt: "Spring batch step flow"

categories:
- Spring batch

tags:
- Spring batch
- Step

toc: true

toc_sticky: true

toc_label: "Spring batch"

last_modified_at: 2022-11-04T09:59:00-05:00

---

## Controlling Step Flow

- 하나의 스텝에서 다른 스텝으로 이동할 때, 흐름을 제어하는 방법
- 스텝의 실패가 반드시 Job의 실패를 의미하지 않는다
- 스텝의 성공의 의미에 추가적인 여러 의미를 부여할  수 있음
- 스텝의 구성에 따라서, 어떤 스텝은 전혀 실행되지 않을 수 있음

---

## 연속 흐름(Sequential flow)

~~~java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
				.start(stepA())
				.next(stepB())
				.next(stepC())
				.build();
}
~~~

- 일반적으로 가장 많이 쓰는 형태
- stepA, stepB, stepC가 순차적으로 발생
- stepA가 실패하면, 전체 잡은 실패가 되고, stepB를 실행하지 않는다.

---

## 조건부 흐름(Conditional flow)

- 위의 연속적 흐름에서 두 가지 결론이 있다.
  - 이전 스텝이 성공하면, 다음 스텝은 반드시 실행된다.
  - 이전 스텝이 실패하면, 잡도 실패해야 한다

- 많은 경우에, 이것은 충분하지만 실패를 처리하기 위한 다양한 스텝을 만들 수 있다.
  - StepA가 성공하면, StepB
  - StepA가 실패하면, StepC

~~~java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
				.start(stepA())
				.on("*").to(stepB())
				.from(stepA()).on("FAILED").to(stepC())
				.end()
				.build();
}
~~~

- on("*")은 ExitStatus를 매칭하기 위한 method
  - "*" : 모든 문자(Regular expression)
- 만약에 on() 에서 다룰 수 없는, ExitStatus가 발생하면 프레임워크는 예외를 던지고, 잡을 실패시킨다.
- on의 적용 방식은, 가장 상세화되어 있는게 우선이다.
  - "*"가 있다고 하더라도, "FAILED"가 상세화되어 있으므로, "FAILED"가 발생하면 "FAILED" 상태

---

## Batch Status Vs Exit Status

- 조건부 흐름에서는, BatchStatus와 ExitStatus를 이해하는게 가장 중요
  - BatchStatus는 JobExecution과 StepExecution을 모두 사용할 수 있는 enumeration
  - BatchStatus : COMPLETED, STARTING, STARTED, STOPPING, STOPPED, FAILED, ABANDONED, or UNKNOWN

~~~java
...
.from(stepA()).on("FAILED").to(stepB())
...
~~~

- "FAILED"는 BatchStatus가 아니라, 스텝 실행이 끝나고 상태를 표시하는 ExitStatus이다.
- 일반적으로는 BatchStatus = ExitStatus이지만, 하지만

~~~java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
			.start(step1()).on("FAILED").end()
			.from(step1()).on("COMPLETED WITH SKIPS").to(errorPrint1())
			.from(step1()).on("*").to(step2())
			.end()
			.build();
}
~~~

- on은 Enumeration(Batch Status)이 아닌 ExitStatus이므로 원하는대로 상태를 표기할 수 있음

~~~java
public class SkipCheckingListener extends StepExecutionListenerSupport {
    public ExitStatus afterStep(StepExecution stepExecution) {
        String exitCode = stepExecution.getExitStatus().getExitCode();
        if (!exitCode.equals(ExitStatus.FAILED.getExitCode()) &&
              stepExecution.getSkipCount() > 0) {
            return new ExitStatus("COMPLETED WITH SKIPS");
        }
        else {
            return null;
        }
    }
}
~~~

- SkipCheckingListener를 통해서, 특정 상태에 따라서 ExitStatus를 다르게 줄 수 있음
  - BatchStatus는 Enumeration이기 때문에, COMPLETED만 표현 가능한 한계를 ExitStatus로 극복 

---

## 중지 구성

~~~java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
				.start(step1())
				.build();
}
~~~

- 스텝만 실행시키고 어떠한 transitions도 없는 경우
  - 스텝이 만약에 ExitStatus가 FAILED라면, Job의 ExitStatus, BatchStatus 모두 FAILED
  - ExitStatus가 COMPLETED이라면, Job의 ExitStatus, BatchStatus도 COMPLETED
- 중지된 스텝은, 잡의 어떤 스텝들의 BatchStatus나 ExitStatus에 영향을 주지 않음
- 중지된 스텝은 오직 Job의 마지막 상태에만 영향을 준다

---

## 스텝에서 종료시키기
- 잡이 BatchStatus가 COMPLETED 상태가 된다면, 다시 재실행할 수 없다
- end() 메소드는 이러한 역할을 하는데 도와준다.
  - end() 메소드의 파라미터에 어떠한 값도 정의하지 않는다면, COMPLETED가 default로 들어감

~~~java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
				.start(step1())
				.next(step2())
				.on("FAILED").end()
				.from(step2()).on("*").to(step3())
				.end()
				.build();
}
~~~

- step2가 실패하면
  - Job 중지
  - BatchStatus와 ExitStatus는 COMPLETED 상태가 되므로 재실행 불가

---

## 스텝 실패시키기

- 스텝을 실패시키면, Job은 BatchStatus가 FAILED 되고 재시작은 할 수 있음
- step2가 실패하고, BatchStatus는 FAILED를 가지면 step3은 실행할 수 없음

~~~java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
			.start(step1())
			.next(step2()).on("FAILED").fail()
			.from(step2()).on("*").to(step3())
			.end()
			.build();
}
~~~

## 주어진 스텝에서 잡을 중지시키기

- 특정 스텝에서 잡을 중지시킨다는건, 잡의 BatchStatus가 STOPPED가 되는걸 의미
- 잡을 중지시키는건, 일시적으로 처리를 멈추고 잡을 재수행하기전에 어떠한 작업을 수행할 수 있음
- step1이 COMPLETED이 되면, Job을 COMPLETE 상태로 종료시키고, 해당 잡을 재수행시에, step2부터 실행

~~~java
@Bean
public Job job() {
	return this.jobBuilderFactory.get("job")
			.start(step1()).on("COMPLETED").stopAndRestart(step2())
			.end()
			.build();
}
~~~





---
## Reference

- [Spring batch docs](https://docs.spring.io/spring-batch/docs/current/reference/html/step.html#controllingStepFlow)
