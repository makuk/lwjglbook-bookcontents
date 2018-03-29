# 게임 루프

이번 장에서는 게임 루프를 만듦으로써 게임 엔진 개발을 본격적으로 시작해 보겠습니다. 게임 루프는 모든 게임에서의 핵심 요소입니다. 사용자 입력 처리, 게임 상태 업데이트, 화면 렌더링 같은 일을 끝없이 반복하죠.

게임 루프의 구조를 아래 코드를 통해 살펴봅시다.

```java
while (keepOnRunning) {
    handleInput();
    updateGameState();
    render();
}
```

이렇게 하면 게임 루프는 끝났다고 할 수 있을까요? 그렇지 않습니다. 위 코드에는 많은 문제점이 있습니다. 먼저, 이 코드는 돌아가는 기기에 따라 속도가 천차만별일 겁니다. 속도가 너무 빠르면 사용자가 게임에서 뭔 일이 일어나는지 알 수 없는 일이 발생하게 될 수도 있습니다. 게다가 이 코드는 기기 자원을 너무 낭비합니다.

그러므로 돌아가는 기기와는 상관없도록 게임 루프가 돌아갈 일정한 주기를 정해 주어야 합니다. 주기를 초당 50 프레임\(Frames Per Second, FPS\)으로 정해 봅시다. 그러면 코드가 이렇게 됩니다.

```java
double secsPerFrame = 1.0d / 50.0d;

while (keepOnRunning) {
    double now = getTime();
    handleInput();
    updateGameState();
    render();
    sleep(now + secsPerFrame – getTime());
}
```

이번 게임 루프는 간단하고 게임에 넣을 수 있을 정도지만 아직도 문제점이 남아 있습니다. 이 코드는 업데이트와 렌더링 메소드가 50 FPS\(`secsPerFrame`은 20ms\)에 정확히 맞게 돌아갈 거라고 가정해 버립니다.

게다가 게임 루프보다 우선 순위가 높은 일이 있어 게임 루프가 돌아가는 걸 방해한다면 게임 상태 업데이트가 불규칙적으로 이루어지게 될 겁니다. 게임 물리학에는 매우 좋지 못한 방법이죠.

마지막으로, 슬립하는 시간이 1/10초정도 오차가 날 수 있습니다. 그러니까 업데이트와 렌더링 메소드가 작동되는 데 시간이 전혀 걸리자 않는다고 하더라도 일정한 프레임 속도를 유지할 수가 없게 되는 것이죠. 보시다시피 이런 문제들은 그리 간단한 게 아닙니다.

인터넷에는 매우 많고 다양한 게임 루프가 있습니다. 이 책에서는 다양한 상황에서 잘 작동되고 그리 복잡해지지 않는 쪽으로 접근해 볼 겁니다. 그럼 여기에서 사용할 게임 루프의 기본 원리를 설명하겠습니다. 이 방식의 이름은 Fixed Step Game Loop입니다.

먼저 게임 상태가 업데이트되는 때와 화면에 게임이 렌더링되는 때를 따로 관리할 수 있어야 합니다. 게임 상태를 일정한 시간마다 업데이트하는 건 중요하기 때문이죠. 특히 물리엔진을 사용한다면 더욱 중요합니다. 한편 렌더링이 제 시간에 끝나지 못하고 다음 루프에서 밀린 렌더링을 처리하는 것도 좋지 못합니다. 프레임을 스킵할 수 있게 하여 유연성을 늘리는 것이 좋겠죠.

그럼 코드를 살펴 보도록 합시다.

```java
double secsPerUpdate = 1.0d / 30.0d;
double previous = getTime();
double steps = 0.0;
while (true) {
  double loopStartTime = getTime();
  double elapsed = loopStartTime - previous;
  previous = current;
  steps += elapsed;

  handleInput();

  while (steps >= secsPerUpdate) {
    updateGameState();
    steps -= secsPerUpdate;
  }

  render();
  sync(current);
}
```

이 게임 루프에서는 스텝을 고정시켜 게임을 업데이트합니다. 그런데 끝없이 렌더링함으로써 컴퓨터 자원을 다 써버리지 않게 하기 위해서는 어떻게 해야 할까요? sync 메소드를 만들어서 이를 해결해 봅시다.

```java
private void sync(double loopStartTime) {
   float loopSlot = 1f / 50;
   double endTime = loopStartTime + loopSlot; 
   while(getTime() < endTime) {
       try {
           Thread.sleep(1);
       } catch (InterruptedException ie) {}
   }
}
```

위 메소드는 어떤 일을 하는 걸까요? 요약하자면 게임 루프가 한 번 돌아가는 시간을 계산하고\(`loopSlot` 변수에 저장됨\) 그 시간까지 남은 만큼 기다려 줍니다. 하지만 그 남은 시간을 한번에 기다리는 대신 조금씩 기다립니다. 이렇게 하면 다른 작업들이 돌아갈 수 있게 되고, 앞서 말했던 정확도 문제도 해결할 수 있습니다. 그리고 그 다음에 해야 할 일들은 이렇습니다.
1.    wait 메소드를 끝내고 다음 루프로 넘어갈 시간을 계산한다\(`endTime` 변수에 저장됨\).
2.    현재 시간과 끝 시간을 비교하고 아직 끝 시간에 도달하지 않았으면 1밀리초를 쉰다.

이제 게임 엔진을 만들기 위해 코드의 기반을 구성해야 합니다. 하지만 그 전에, 렌더링 주기를 조절하기 위한 다른 방법에 대해 이야기해 봅시다.위 코드에 나와있듯이 기다릴 시간을 조절하기 위해 조금씩 슬립을 했는데, 프레임 속도를 제한하기 위해 다른 방식으로도 접근해 볼 후 있습니다. 수직 동기화\(v-sync, vertical synchronization\)를 사용하는 것입니다. 수직 동기화의 주된 목적은 스크린 테어링\(screen tearing\)을 방지하는 것입니다. 스크린 테어링이란 렌더링하는 중에 영상 메모리가 업데이트 될 때 생기는 현상입니다. 일부는 이전 화면이, 일부는 다음 화면이 보여지게 되는 것이죠. 수직 동기화를 켜 놓으면 화면이 렌더링되는 동안에는 GPU에 다음 화면이 보내지지 않게 됩니다.

수직 동기화를 켜면 비디오 카드의 주사율이 동기화되고, 일정하게 프레임 속도를 유지시킬 수 있게 됩니다. 코드를 이렇게 짜면 됩니다.

```java
glfwSwapInterval(1);
```

이렇게 씀으로써 최소 화면 업데이트를 하나는 기다려야 화면을 그릴 수 있도록 정할 수 있습니다. 사실 화면을 직접적으로 그리는 건 아닙니다. 대신 정보를 버퍼에 저장하고 그걸 이 메소드를 호출하여 전환합니다.

```java
glfwSwapBuffers(windowHandle);
```

그러므로 수직 동기화를 사용하면 시간을 확인하기 위해 조금씩 슬립할 필요가 없습니다. 게다가 프레임 속도가 그래픽 카드의 주사율과 동일해질 것입니다. 그렇다는 건, 만약 그래픽 카드가 60Hz\(초당 60번\)으로 정해져 있다면 60FPS가 될 것입니다. `glfwSwapInterval` 메소드의 파라미터 숫자를 1보다 높게 잡으면 속도를 늦출 수 있습니다\(2로 하면 30FPS가 되겠죠\).

그럼 다시 돌아가서 코드를 재구성해 봅시다. 파라미터로 기본 설정\(이름과 크기같은 것들\)을 받는 `Window` 클래스를 만들고 모든 GLFW 창 초기화 코드를 그 안에 캡슐화시킬 겁니다. 이 `Window` 클래스에는 게임 루프에 쓰일 키 입력 감지 메소드도 들어갑니다.

```java
public boolean isKeyPressed(int keyCode) {
    return glfwGetKey(windowHandle, keyCode) == GLFW_PRESS;
}
```

이 `Window` 클래스는 창을 초기화하는 일을 하는 동시에 창 크기가 재조정되는지도 알아야 합니다. 그러므로 창 크기가 재조정될 때 호출될 메소드를 만들어야겠죠. 그 메소드는 프레임 버퍼\(렌더링되는 영역, 이 예제에서는 화면 영역\)의 너비와 높이의 픽셀 값을 받게 될 겁니다. 화면 좌표로 프레임 버퍼의 너비와 높이를 받고 싶다면 `glfwSetWindowSizeCallback` 메소드를 사용해도 됩니다. 화면 좌표는 꼭 픽셀과 대응되는 것은 아닙니다\(레티나 디스플레이를 사용하는 맥이 이에 해당합니다\). 너비와 높이 정보는 오픈GL을 사용하는 데에 쓸 것이므로 화면상의 좌표가 아니라 픽셀 값이 필요합니다. GLFW doc에서 자세한 정보를 알아볼 수 있습니다.

```java
// Setup resize callback
glfwSetFramebufferSizeCallback(windowHandle, (window, width, height) -> {
    Window.this.width = width;
    Window.this.height = height;
    Window.this.setResized(true);
});
```

게임 렌더링 로직을 관리하기 위한 `Renderer` 클래스도 만들고 그 안에는 비어있는 `init` 메소드와 설정된 색으로 화면을 지울 메소드를 만들겠습니다.

```java
public void init() throws Exception {
}

public void clear() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
}
```

그리고 게임 로직을 캡슐화할 `IGameLogic` 인터페이스도 만들겠습니다. 이걸 만듦으로써 이 게임 엔진을 다른 게임에도 재사용할 수 있게 될 것입니다. 이 인터페이스에는는 입력을 얻어오는 메소드, 게임 상태를 업데이트하는 메소드, 게임의 특정 데이터를 렌더링하는 메소드를 넣을 겁니다.

```java
public interface IGameLogic {

    void init() throws Exception;

    void input(Window window);

    void update(float interval);

    void render(Window window);
}
```

그리고 게임 루프 코드를 넣을 `GameEngine` 클래스도 만들겠습니다. 이 클래스는 다른 스레드에서 돌아가게 하기 위해 `Runnable` 인터페이스를 구현하도록 하겠습니다.

```java
public class GameEngine implements Runnable {

    //..[Removed code]..

    private final Thread gameLoopThread;

    public GameEngine(String windowTitle, int width, int height, boolean vsSync, IGameLogic gameLogic) throws Exception {
        gameLoopThread = new Thread(this, "GAME_LOOP_THREAD");
        window = new Window(windowTitle, width, height, vsSync);
        this.gameLogic = gameLogic;
        //..[Removed code]..
    }
```

`vSync` 파라미터로는 수직 동기화 사용 여부를 결정할 수 있습니다. 이제 게임 루프가 들어 있는 `GameEngine` 클래스의 run 메소드를 실행할 스레드를 새로 하나 만들어봅시다. 

```java
public void start() {
    gameLoopThread.start();
}

@Override
public void run() {
    try {
        init();
        gameLoop();
    } catch (Exception excp) {
        excp.printStackTrace();
    }
}
```

`GameEngine` 클래스의 스레드를 시작하는 start 메소드가 비동기적으로 run 메소드를 실행시키게 됩니다. 이 메소드는 초기화하는 일과 창이 닫힐 때까지 게임 루프를 돌리는 일을 할 겁니다. 나중에 업데이트될 GLFW를 같은 스레드에서 초기화하는 것이 매우 중요합니다. 그러므로 `init` 메소드에는 Window와 `Renderer` 인스턴스를 초기화하는 코드를 넣겠습니다.

이 코드에서는 Timer\(경과된 타임을 계산하는 유틸리티 메소드가 있는 클래스\)같은 보조 클래스들이 몇 개 들어가 있고, 게임 루프 로직에 사용될 것입니다.

`GameEngine` 클래스에 있는 input과 update 메소드를 `IGameLogic` 인스턴스에 넘겨줍니다. render 메소드도 IGameLogic 인스턴스에 넘겨준 뒤 창을 업데이트합니다.

```java
protected void input() {
    gameLogic.input(window);
}

protected void update(float interval) {
    gameLogic.update(interval);
}

protected void render() {
    gameLogic.render(window);
    window.update();
}
```

프로그램의 시작점이 되는 main 메소드를 포함하는 클래스는 `GameEngine` 인스턴스를 만들고 시작시킵니다.

```java
public class Main {

    public static void main(String[] args) {
        try {
            boolean vSync = true;
            IGameLogic gameLogic = new DummyGame();
            GameEngine gameEng = new GameEngine("GAME",
                600, 480, vSync, gameLogic);
            gameEng.start();
        } catch (Exception excp) {
            excp.printStackTrace();
            System.exit(-1);
        }
    }

}
```

마지막으로 게임 로직 클래스를 만들면 됩니다. 이번 장에서는 간단하게 만들겠습니다. 사용자가 위/아래 화살표를 누름에 따라 창의 초기화 색을 증가/감소시킵니다. render 메소드는 창을 그 색으로 초기화하는 일만 합니다.

```java
public class DummyGame implements IGameLogic {

    private int direction = 0;

    private float color = 0.0f;

    private final Renderer renderer;

    public DummyGame() {
        renderer = new Renderer();
    }

    @Override
    public void init() throws Exception {
        renderer.init();
    }

    @Override
    public void input(Window window) {
        if ( window.isKeyPressed(GLFW_KEY_UP) ) {
            direction = 1;
        } else if ( window.isKeyPressed(GLFW_KEY_DOWN) ) {
            direction = -1;
        } else {
            direction = 0;
        }
    }

    @Override
    public void update(float interval) {
        color += direction * 0.01f;
        if (color > 1) {
            color = 1.0f;
        } else if ( color < 0 ) {
            color = 0.0f;
        }
    }

    @Override
    public void render(Window window) {
        if ( window.isResized() ) {
            glViewport(0, 0, window.getWidth(), window.getHeight());
            window.setResized(false);
        }
        window.setClearColor(color, color, color, 0.0f);
        renderer.clear();
    }    
}
```

`render` 메소드에서는 창 크기가 변했는지 확인하고 뷰포트 위치를 창의 중앙으로 바꿉니다.

방금 짠 클래스 체계는 게임 엔진 코드와 게임 코드를 분리시키는 데에 도움이 될 겁니다. 지금으로서는 필요 없어 보이지만, 게임 엔진을 재사용하기 위해서는 모든 게임이 사용하는 상태 로직같은 일반적인 기능들과 특정 게임에서 사용하는 텍스쳐나 리소스 등을 분리해야 합니다. 게임 엔진 코드가 점점 더 복잡해지게 되므로 후에 클래스 체계를 재구성할 겁니다.

## 스레드 문제

OSX에서 지금까지 나온 코드를 실행하면 아래같은 에러가 날 겁니다.

```
Exception in thread "GAME_LOOP_THREAD" java.lang.ExceptionInInitializerError
```

이게 무슨 말일까요? GLFW 라이브러리의 일부 기능은 메인 `스레드` 이외의 `스레드`에서 실행될 수 없습니다. 지금 여기에서는 `GameEngine 클래스`의 `init` 메소드에서 창 생성을 포함한 초기화 작업을 하고 있습니다. 그 메소드는 같은 클래스의 `run` 메소드에서 실행되고, 그 `run` 메소드는 프로그램이 실행될 때의 `스레드`가 아닌 새 `스레드`에서 호출됩니다.

이건 GLFW 라이브러리에서의 필수사항이고 게임 루프용의 새 스레드를 생성하지 말아야 한다는 것을 암시합니다. 그럼 창 관련된 것들은 메인 스레드에서 실행시켜야 하는데, 그럼 렌더링이 되지 않을 겁니다. 오픈GL은 그 context가 생성된 것과 같은 `스레드`에서 실행되어야 하기 때문입니다.

윈도우와 리눅스에서는 메인 스레드에서 GLFW를 초기화하지 않아도 예제 코드가 작동되겠지만, OSX에서는 아닙니다. 그래서 OSX에서도 작동이 되게 하기 위해서는 `GameEngine` 클래스에 있는 `run` 메소드의 코드를 약간 수정해야 합니다.

```java
public void start() {
    String osName = System.getProperty("os.name");
    if ( osName.contains("Mac") ) {
        gameLoopThread.run();
    } else {
        gameLoopThread.start();
    }
}
```

위 코드는 OSX일 경우에 게임 루프 스레드를 만들지 않고 메인 스레드에서 게임 루프 코드를 실행시킵니다. 이건 완벽한 해결책은 아니지만 맥에서 예제 코드를 작동시킬 수는 있습니다. 포럼에 나오는 다른 해결책\(JVM을 `-XstartOnFirstThread` 플래그를 넣어서 실행시키는 것\)은 잘 먹히지 않는 것으로 보입니다.

나중에는 LWJGL이 다른 GUI 라이브러리에게 이러한 제한 사항을 적용하는지를 보기 위해 그런 라이브러리를 제공하는지 알아보는 것도 재미있을 겁니다.  \(Timo Bühlmann님께 이 점을 지적해주신 것에 대해 큰 감사를 표합니다.\).

## 플랫폼 차이 \(OSX\)

지금까지 나온 코드는 윈도우와 리눅스에서 작동시킬 수 있지만, OSX에서는 몇 줄을 더 추가해야 합니다. GLFW doc에서 이렇게 서술되어 있습니다.

> OS X에서 유일하게 지원하는 오픈GL 3.x와 4.x의 context는 상위 호환되는 핵심 프로필 context입니다. 10.7 Lion은 3.2를, 10.9 Mavericks에서는 3.3과 4.1을 지원합니다. GPU는 context 생성을 성공적으로 하기 위해 지정된 오픈GL 버전을 모든 경우에 지원해야 합니다.
그러므로, 나중에 나올 기능들을 사용하기 위해서는 창을 생성하기 전에 `window` 클래스에 이 코드를 넣어야 합니다.

```java
        glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
        glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 2);
        glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
        glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
```

이 코드를 쓰면 프로그램이 3.2와 4.1 사이에서 지원되는 가장 높은 오픈GL 버전을 사용하도록 할 수 있습니다. 이 코드가 없으면 옛날 버전의 오픈GL이 쓰일 겁니다.

