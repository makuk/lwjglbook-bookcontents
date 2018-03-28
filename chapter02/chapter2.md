# The Game Loop

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

위 메소드는 어떤 일을 하는 걸까요? 요약하자면 게임 루프가 한 번 돌아가는 시간을 게산하고\(`loopSlot` 변수에 저장됨\) 그 시간까지 남은 만큼 기다려 줍니다. 하지만 그 남은 시간을 한번에 기다리는 대신 조금씩 기다립니다. 이렇게 하면 다른 작업들이 돌아갈 수 있게 되고, 앞서 말했던 정확도 문제도 해결할 수 있습니다. 그리고 그 다음에 해야 할 일들은 이렇습니다.
1.    Calculate the time at which we should exit this wait method and start another iteration of our game loop \(which is the variable `endTime`\).  
2.    Compare the current time with that end time and wait just one millisecond if we have not reached that time yet.

Now it is time to structure our code base in order to start writing our first version of our Game Engine. But before doing that we will talk about another way of controlling the rendering rate. In the code presented above, we are doing micro-sleeps in order to control how much time we need to wait. But we can choose another approach in order to limit the frame rate. We can use v-sync \(vertical synchronization\). The main purpose of v-sync is to avoid screen tearing. What is screen tearing? It’s a visual effect that is produced when we update the video memory while it’s being rendered. The result will be that part of the image will represent the previous image and the other part will represent the updated one. If we enable v-sync we won’t send an image to the GPU while it is being rendered onto the screen.

When we enable v-sync we are synchronizing to the refresh rate of the video card, which at the end will result in a constant frame rate. This is done with the following line:

```java
glfwSwapInterval(1);
```

With that line we are specifying that we must wait, at least, one screen update before drawing to the screen. In fact, we are not directly drawing to the screen. We instead store the information to a buffer and we swap it with this method:

```java
glfwSwapBuffers(windowHandle);
```

So, if we enable v-sync we achieve a constant frame rate without performing the micro-sleeps to check the available time. Besides that, the frame rate will match the refresh rate of our graphics card. That is, if it’s set to 60Hz \(60 times per second\), we will have 60 Frames Per Second. We can scale down that rate by setting a number higher than 1 in the `glfwSwapInterval` method \(if we set it to 2, we would get 30 FPS\).

Let’s get back to reorganize the source code. First of all we will encapsulate all the GLFW Window initialization code in a class named `Window` allowing some basic parameterization of its characteristics \(such as title and size\). That `Window` class will also provide a method to detect key presses which will be used in our game loop:

```java
public boolean isKeyPressed(int keyCode) {
    return glfwGetKey(windowHandle, keyCode) == GLFW_PRESS;
}
```

The `Window` class besides providing the initialization code also needs to be aware of resizing. So it needs to setup a callback that will be invoked whenever the window is resized. The callback will receive the width and height, in pixels, of the framebuffer \(the rendering area, in this sample, the display area\). If you want the width, height of the framebuffer in screen coordinates you may use the the  `glfwSetWindowSizeCallback`method. Screen coordinates don't necessarilly correspond to pixels \(for instance, on a Mac with Retina display. Since we are going to use that information when performing some OpenGL calls, we are interested in pixels not in screen coordinates. You can get more infomation in the GLFW documentation.

```java
// Setup resize callback
glfwSetFramebufferSizeCallback(windowHandle, (window, width, height) -> {
    Window.this.width = width;
    Window.this.height = height;
    Window.this.setResized(true);
});
```

We will also create a `Renderer` class which will handle our game render logic. By now, it will just have an empty `init` method and another method to clear the screen with the configured clear color:

```java
public void init() throws Exception {
}

public void clear() {
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
}
```

Then we will create an interface named `IGameLogic` which will encapsulate our game logic. By doing this we will make our game engine reusable across different titles. This interface will have methods to get the input, to update the game state and to render game-specific data.

```java
public interface IGameLogic {

    void init() throws Exception;

    void input(Window window);

    void update(float interval);

    void render(Window window);
}
```

Then we will create a class named `GameEngine` which will contain our game loop code. This class will implement the `Runnable` interface since the game loop will be run inside a separate thread:

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

The `vSync` parameter allows us to select if we want to use v-sync or not. You can see we create a new Thread which will execute the run method of our `GameEngine` class which will contain our game loop:

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

Our `GameEngine` class provides a start method which just starts our Thread so the run method will be executed asynchronously. That method will perform the initialization tasks and will run the game loop until our window is closed. It is very important to initialize GLFW inside the thread that is going to update it later. Thus, in that `init` method our Window and `Renderer` instances are initialized.

In the source code you will see that we created other auxiliary classes such as Timer \(which will provide utility methods for calculating elapsed time\) and will be used by our game loop logic.

Our `GameEngine` class just delegates the input and update methods to the `IGameLogic` instance. In the render method it delegates also to the `IGameLogic`  instance and updates the window.

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

Our starting point, our class that contains the main method will just only create a `GameEngine` instance and start it.

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

At the end we only need to create or game logic class, which for this chapter will be a simpler one. It will just increase / decrease the clear color of the window whenever the user presses the up / down key. The render method will just clear the window with that color.

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

In the `render` method we get notified when the window has been resized in order to update the viewport to locate the center of the coordinates to the center of the window.

The class hierarchy that we have created will help us to separate our game engine code from the code of a specific game. Although it may seem unnecessary at this moment, we need to isolate generic tasks that every game will use from the state logic, artwork and resources of a specific game in order to reuse our game engine. In later chapters we will need to restructure this class hierarchy as our game engine gets more complex.

## Threading issues

If you try to run the source code provided above in OSX you will get an error like this:

```
Exception in thread "GAME_LOOP_THREAD" java.lang.ExceptionInInitializerError
```

What does this mean? The answer is that some functions of the GLFW library cannot be called in a `Thread` which is not the main `Thread`. We are doing the initializing stuff, including window creation in the `init` method of the  `GameEngine class`. That method gets called in the `run` method of the same class, which is invoked by a new `Thread` instead of the one that's used to launch the program.

This is a constraint of the GLFW library and basically it implies that we should avoid the creation of new Threads for the game loop. We could try to create all the Windows related stuff in the main thread but we will not be able to render anything. The problem is that, OpenGL calls need to be performed in the same `Thread` that its context was created.

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

In the future it may be interesting to explore if LWJGL provides other GUI libraries to check if this restriction applies to them. \(Many thanks to Timo Bühlmann for pointing out this issue\).

## 플랫폼 차이 \(OSX\)

지금까지 나온 코드는 윈도우와 리눅스에서 작동시킬 수 있지만, OSX에서는 몇 줄을 더 추가해야 합니다. GLFW doc에서 이렇게 서술되어 있습니다.

> The only OpenGL 3.x and 4.x contexts currently supported by OS X are forward-compatible, core profile contexts. The supported versions are 3.2 on 10.7 Lion and 3.3 and 4.1 on 10.9 Mavericks. In all cases, your GPU needs to support the specified OpenGL version for context creation to succeed.

그러므로, 나중에 나올 기능들을 사용하기 위해서는 창을 생성하기 전에 `window` 클래스에 이 코드를 넣어야 합니다.

```java
        glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
        glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 2);
        glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
        glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
```

이 코드를 쓰면 프로그램이 3.2와 4.1 사이에서 가장 높은 오픈GL 버전을 사용하도록 할 수 있습니다. 이 코드가 없으면 옛날 버전의 오픈GL이 쓰일 겁니다.

