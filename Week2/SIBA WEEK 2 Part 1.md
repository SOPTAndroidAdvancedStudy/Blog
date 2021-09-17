

메인 스레드와 Handler Part1 - Handler와 Looper 그리고 MessageQueue의 동작 방식 

-  Handler와 Looper 그리고 MessageQueue가 어떻게 동작하는지 안드로이드 프레임 워크를 통해 알아봅니다. 

안드로이드의 메인스레드인 ActivityThread.java 를 이해하는데에 큰 배경 지식이 된다는 점에서 Handler와 Looper 그리고 MessageQueue의 동작 방식을 이해하는 것은 큰 가치가 있는 일입니다. 스레드 통신과 약간의 자료구조에 대한 부분까지 생각해보는 좋은 기회가 될 것입니다. 따라서 이번 아티클에서는 안드로이드 프레임워크의 코드와 함께 Handler와 Looper 그리고 MessageQueue의 동작 방식에 대해 깊게 알아보겠습니다. 

Looper 

Message와 MessageQueue

Handler

안드로이드 Thread 모델은 자바의 Thread 모델을 그대로 따른다.

출처: https://link2me.tistory.com/1233 [소소한 일상 및 업무TIP 다루기]

소결 