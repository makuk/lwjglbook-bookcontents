# 첫걸음

이 책에서는 3D 게임 개발에 관한 주요 기술이 나옵니다. 자바와 LightWeight Java Game Library\([LWJGL](http://www.lwjgl.org/)\)를 사용하여 개발할 겁니다. LWJGL 라이브러리는 OpenGL같은 저수준 API\(Application Programming Interface, 응용 프로그램 프로그래밍 인터페이스\)에 접근할 수 있게 해 줍니다.

LWJGL은 OpenGL의 포장지 같은 역할을 하는 저수준 API입니다. 만약 짧은 기간에 3D 게임을 개발하고 싶으시다면 \[JmonkeyEngine\]같은 다른 것을 알아보시는 게 좋을 겁니다. 이 저수준 API를 사용하게 되면 결과를 보기 위해서는 많은 개념을 배우고 많은 코드를 써야 합니다. 이것의 장점은 3D 그래픽에 대해서 더 잘 이해할 수 있고 또한 이런 것들을 더욱 능숙하게 다룰 수 있게 된다는 것입니다.

앞에서 말했듯이 이 책에서는 자바를 사용할 겁니다. 자바 10을 사용할 것이니 오라클에서 자바 SDK를 다운로드해야 합니다. 운영체제에 맞는 설치기를 골라서 설치하기만 하면 됩니다. 이 책은 당신이 자바를 어느 정도 다룰 수 있다는 가정하에 진행됩니다.

예제를 실행하기 위해서 자신이 원하는 자바 IDE를 사용해도 됩니다. 자바 10 지원이 잘 되어 있는 IntelliJ IDEA를 써도 좋습니다. 현재 자바 10은 64비트 플랫폼에서만 사용할 수 있으므로 IntelliJ 64비트 버전을 받아야 합니다. IntelliJ는 무료 오픈소스인 커뮤니티 버전을 제공하고 있고, 다음 링크에서 받으실 수 있습니다. [https://www.jetbrains.com/idea/download/](https://www.jetbrains.com/idea/download/ "Intellij").

![](/chapter01/intellij.png)

예제를 빌드하기 위해서 [Maven](https://maven.apache.org/)을 사용할 겁니다. Maven은 대부분의 IDE에 들어 있으므로 그 안에서 예제를 바로 여시면 됩니다. 예제가 들어있는 폴더를 열기만 하면 IntelliJ가 그게 Maven 프로젝트라는걸 감지할 겁니다.

![](/chapter01/maven_project.png)

Maven은 프로젝트의 의존성\(프로젝트에 사용되는 라이브러리\)과 빌드 프로세스 중에 시행해야 할 단계에 대한 정보를 관리하는 `pom.xml` \(프로젝트 객체 모델, Project Object Model\)라는 XML 파일을 기반으로 프로젝트를 빌드합니다. Maven은 CoC\(Convention over Configuration\)를 따릅니다. 즉, 표준 프로젝트 구조와 네이밍 규칙을 따르면 소스 파일의 위치와 컴파일된 클래스가 위치할 곳을 따로 명시하지 않아도 됩니다.

이 책은 Maven 강좌를 하는 책이 아니기 때문에 더 자세한 정보가 필요하다면 인터넷을 찾아보시기 바랍니다. 소스 코드 폴더는 사용되는 플러그인을 정의하고 사용된 라이브러리의 버전을 수집하는 상위 프로젝트를 정의합니다.

LWJGL 3.1에서는 프로젝트가 빌드되는 방법이 약간 바뀌었습니다. 기본 코드는 더욱 모듈식이 되었으며 큰 jar 파일을 통째로 넣는 대신 자신이 필요한 패키지만 넣을 수 있게 되었습니다. 그래서 의존성을 하나하나 지정해야 하는데, [다운로드](https://www.lwjgl.org/download) 페이지에는 pom 파일을 생성해 주는 스크립트가 들어가 있습니다. 이 강좌에서는 GLFW와 오픈GL이 들어있는 걸로 하겠습니다. 코드를 열어서 pom 파일이 어떻게 생겼는지 확인해 볼 수도 있습니다.

LWJGL 플랫폼 의존성은 네이티브 라이브러리를 자동으로 압축 해제해주므로 `mavennatives`같은 다른 플러그인을 깔 필요가 없습니다. LWJGL 플랫폼을 구성할 때 쓰이는 프로퍼티를 설정하기 위해 프로필 세 개를 설정하면 됩니다. 그 프로필은 각각 윈도우, 리눅스, 맥 이 세 개의 프로퍼티를 각각 설정할 겁니다.

```xml
    <profiles>
        <profile>
            <id>windows-profile</id>
            <activation>
                <os>
                    <family>Windows</family>
                </os>
            </activation>
            <properties>
                <native.target>natives-windows</native.target>
            </properties>                
        </profile>
        <profile>
            <id>linux-profile</id>
            <activation>
                <os>
                    <family>Linux</family>
                </os>
            </activation>
            <properties>
                <native.target>natives-linux</native.target>
            </properties>                
        </profile>
        <profile>
            <id>OSX-profile</id>
            <activation>
                <os>
                    <family>mac</family>
                </os>
            </activation>
            <properties>
                <native.target>natives-osx</native.target>
            </properties>
        </profile>
    </profiles>
```

각각의 프로젝트 안에서 LWJGL 플랫폼 의존성이 현재 플랫폼에 맞게 프로필 안에 지정된 적절한 프로퍼티를 사용할 겁니다.

```xml
        <dependency>
            <groupId>org.lwjgl</groupId>
            <artifactId>lwjgl-platform</artifactId>
            <version>${lwjgl.version}</version>
            <classifier>${native.target}</classifier>
        </dependency>
```

또한 모든 프로젝트는 실행 가능한 jar \(java -jar jar\_파일\_이름.jar 명령어로 실행 가능한 파일\) 파일을 생성합니다. 올바른 값이 든 `MANIFEST.MF` 파일과 함께 jar 파일을 생성하는 maven-jar-plugin이 이러한 일을 합니다. `MANIFEST.MF` 파일에서의 가장 중요한 속성은 프로그램의 시작점을 설정하는 `Main-Class`입니다. 또한 모든 종속성은 그 파일의 `Class-Path` 속성의 항목으로 들어갑니다. 그걸 다른 컴퓨터에서 실행시키기 위해서는 타겟 디렉토리 안에 있는 메인 jar 파일과 lib 디렉토리를 모든 jar 파일이 들어있는 채로 복사하기만 하면 됩니다.

LWJGL 클래스를 포함하는 jar 파일들은 너이티브 라이브러리도 포함하고 있급니다. LWJGL은 그걸 압축 해제하고 JVM이 라이브러리를 탐색할 경로에 추가하는 일도 담당합니다.

1장의 소스 코드는 LWJGL 사이트 \([http://www.lwjgl.org/guide](http://www.lwjgl.org/guide)\)에서 가져왔습니다. 보시다시피 GUI 라이브러리로 스윙이나 자바FX 같은걸 사용하지 않습니다. 그 대신 오픈GL context가 그대로 들어있는, 창 등의 GUI 구성 요소와 키 입력이나 마우스 움직임 등의 이벤트을 관리하는 [GLFW](www.glfw.org)를 사용합니다. 이전 버전의 LWJGL은 자작 API를 제공했지만 LWJGL 3에서는 GLFW가 창 관리에 선호되는 API입니다.

예제 소스 코드는 잘 문서화되어있고 간단하므로 따로 설명을 달지는 않겠습니다.

환경을 올바르게 설정했다면 실행 시 빨간 배경의 창이 뜰 겁니다.

![Hello World](hello_world.png)

**이 책의 소스코드는 **[**GitHub**](https://github.com/lwjglgamedev/lwjglbook)**에 올라와 있습니다.**
