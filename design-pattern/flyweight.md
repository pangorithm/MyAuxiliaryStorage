### 목적

메모리 사용을 최적화하기 위한 구조적 디자인 패턴으로, 공유를 통해 많은 수의 비슷한 객체를 효율적으로 지원.

객체의 공유 가능한 상태(내부 상태)와 공유할 수 없는 상태(외부 상태)를 분리하여, 공유 가능한 부분을 재사용함으로써 메모리 사용량을 줄인다.

### 타입스크립트 예시 코드

```
interface Flyweight {
    operation(extrinsicState: string): void;
}

class ConcreteFlyweight implements Flyweight {
    private intrinsicState: string;

    constructor(intrinsicState: string) {
        this.intrinsicState = intrinsicState;
    }

    public operation(extrinsicState: string): void {
        console.log(`Flyweight with intrinsic state ${this.intrinsicState} and extrinsic state ${extrinsicState}`);
    }
}

class FlyweightFactory {
    private flyweights: { [key: string]: Flyweight } = {};

    public getFlyweight(key: string): Flyweight {
        if (!(key in this.flyweights)) {
            this.flyweights[key] = new ConcreteFlyweight(key);
        }
        return this.flyweights[key];
    }
}

// 클라이언트 코드
const factory = new FlyweightFactory();
const flyweight1 = factory.getFlyweight('state1'); // 생성
const flyweight2 = factory.getFlyweight('state2'); // 생성
const flyweight3 = factory.getFlyweight('state1'); // 재사용

flyweight1.operation('extrinsicState1');
flyweight2.operation('extrinsicState2');
flyweight3.operation('extrinsicState3');
```

FlyweightFactory는 ConcreteFlyweight 객체의 생성을 관리한다.

클라이언트가 요청할 때, 이미 생성된 인스턴스가 있으면 그 인스턴스를 반환하고, 없으면 새로 생성하여 반환한다.

이 방식으로 ConcreteFlyweight 인스턴스를 재사용함으로써 메모리 사용을 최적화할 수 있다.

flyweight1과 flyweight3은 같은 내부 상태(state1)를 공유하는 동일한 객체임을 주의해야 한다.

### 싱글톤 패턴과의 비교

-   **목적과 적용 분야**: **싱글톤** 패턴은 전역적으로 하나의 인스턴스만을 유지하려는 목적으로 사용되며, 주로 애플리케이션의 설정, 로깅, 데이터베이스 연결과 같은 공유 리소스에 접근할 때 사용됩니다. 반면, **플라이웨이트** 패턴은 메모리 사용량을 줄이기 위해 많은 수의 객체들 사이에서 상태를 공유하고자 할 때 사용됩니다.
-   **인스턴스 관리**: **싱글톤** 패턴에서는 단일 인스턴스가 전역적으로 접근 가능하도록 관리되며, 이 인스턴스는 전체 애플리케이션에서 유일해야 합니다. **플라이웨이트** 패턴에서는 여러 인스턴스가 생성될 수 있지만, 공유 가능한 상태를 가진 인스턴스는 재사용됩니다.
-   **불변성**: **싱글턴** 객체는 변할 수 있습니다 (mutable). **플라이웨이트** 객체들은 변할 수 없다 (immutable).
-   **사용 예**: **싱글톤** 패턴은 데이터베이스 연결이나 로깅 시스템 같은 애플리케이션 전역에서 단 하나만 존재해야 하는 인스턴스에 적합합니다. **플라이웨이트** 패턴은 게임 개발에서 수많은 캐릭터 객체를 효율적으로 관리하거나, 문서 편집 애플리케이션에서 문자 스타일링 객체를 공유하는 데 사용됩니다.

### 추가) 자바 예시코드

타입스크립트 예제의 경우 너무 간단해서 오히려 감이 잡히지 않았다.

비교적 더 어려운 자바 코드 예시를 통해 살펴보도록 한다.

원래대로라면 각 class마다 따로 파일을 생성해주어야 하지만

포스팅 편의를 위해 같은 패키지라면 하나의 코드 블럭에서 확인하도록 한다.

```
package refactoring_guru.flyweight.example.trees;

import java.awt.*;
import java.util.HashMap;
import java.util.Map;

// 각 트리에 대한 고유한 상태를 포함한다.
public class Tree {
    private int x;
    private int y;
    private TreeType type;

    public Tree(int x, int y, TreeType type) {
        this.x = x;
        this.y = y;
        this.type = type;
    }

    public void draw(Graphics g) {
        type.draw(g, x, y);
    }
}

// 여러 트리가 공유하는 상태를 포함
public class TreeType {
    private String name;
    private Color color;
    private String otherTreeData;

    public TreeType(String name, Color color, String otherTreeData) {
        this.name = name;
        this.color = color;
        this.otherTreeData = otherTreeData;
    }

    public void draw(Graphics g, int x, int y) {
        g.setColor(Color.BLACK);
        g.fillRect(x - 1, y, 3, 5);
        g.setColor(color);
        g.fillOval(x - 5, y - 10, 10, 10);
    }
}

// 플라이웨이트 생성의 복잡성을 캡슐화
public class TreeFactory {
    static Map<String, TreeType> treeTypes = new HashMap<>();

    public static TreeType getTreeType(String name, Color color, String otherTreeData) {
        TreeType result = treeTypes.get(name);
        if (result == null) {
            result = new TreeType(name, color, otherTreeData);
            treeTypes.put(name, result);
        }
        return result;
    }
}
```

각 Tree는 좌표와 타입이라는 고유한 값을 갖지만 TreeType은 이름과 색, 나무의 정보를 공유하기도 한다.

따라서 TreeFactory를 통해 TreeType을 맵 객체를 통해 새로 생성하거나 재사용하여 공유한다. 

```
package refactoring_guru.flyweight.example.forest;

import refactoring_guru.flyweight.example.trees.Tree;
import refactoring_guru.flyweight.example.trees.TreeFactory;
import refactoring_guru.flyweight.example.trees.TreeType;

import javax.swing.*;
import java.awt.*;
import java.util.ArrayList;
import java.util.List;

public class Forest extends JFrame {
    private List<Tree> trees = new ArrayList<>();

    public void plantTree(int x, int y, String name, Color color, String otherTreeData) {
        TreeType type = TreeFactory.getTreeType(name, color, otherTreeData);
        Tree tree = new Tree(x, y, type);
        trees.add(tree);
    }

    @Override
    public void paint(Graphics graphics) {
        for (Tree tree : trees) {
            tree.draw(graphics);
        }
    }
}
```

나무를 새로 생성할 때 TreeFactory를 통해 type을 얻어 Tree 객체를 생성할 때 공유해준다.

tree의 draw 매서드를 실행하게 되면 tree의 draw 매서드가 type의 draw를 호출하여 그리게 된다.

(type에 좌표를 매개변수로 넘겨줘 TreeType에서 그리는 것이 아니라 getter를 이용해 tree가 type의 필드를 얻어 그려도 될 것으로 보인다.)

```
package refactoring_guru.flyweight.example;

import refactoring_guru.flyweight.example.forest.Forest;

import java.awt.*;

public class Demo {
    static int CANVAS_SIZE = 500;
    static int TREES_TO_DRAW = 1000000;
    static int TREE_TYPES = 2;

    public static void main(String[] args) {
        Forest forest = new Forest();
        for (int i = 0; i < Math.floor(TREES_TO_DRAW / TREE_TYPES); i++) {
            forest.plantTree(random(0, CANVAS_SIZE), random(0, CANVAS_SIZE),
                    "Summer Oak", Color.GREEN, "Oak texture stub");
            forest.plantTree(random(0, CANVAS_SIZE), random(0, CANVAS_SIZE),
                    "Autumn Oak", Color.ORANGE, "Autumn Oak texture stub");
        }
        forest.setSize(CANVAS_SIZE, CANVAS_SIZE);
        forest.setVisible(true);

        System.out.println(TREES_TO_DRAW + " trees drawn");
        System.out.println("---------------------");
        System.out.println("Memory usage:");
        System.out.println("Tree size (8 bytes) * " + TREES_TO_DRAW);
        System.out.println("+ TreeTypes size (~30 bytes) * " + TREE_TYPES + "");
        System.out.println("---------------------");
        System.out.println("Total: " + ((TREES_TO_DRAW * 8 + TREE_TYPES * 30) / 1024 / 1024) +
                "MB (instead of " + ((TREES_TO_DRAW * 38) / 1024 / 1024) + "MB)");
    }

    private static int random(int min, int max) {
        return min + (int) (Math.random() * ((max - min) + 1));
    }
}
```

플라이웨이트 패턴을 통해 숲 그림을 그리는 예제이다.  
고유한 값의 좌표와 공유하는 값인 이름, 색, 기타 정보를 파라미터로 나무를 생성하는 것을 확인할 수 있다.