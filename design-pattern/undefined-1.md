---
description: 정적 팩토리 메서드와 다릅니다!
---

# 팩토리 메서드 패턴

팩토리 메서드 패턴은

* 인스턴스의 생성 책임을 인터페이스의 추상 메서드로 감싸는 패턴이다.
* 팩토리 인터페이스를 구현하는 각 콘크리트 클래스들이 어떤 타입의 인스턴스를 만들 것인지 결정한다.

이 패턴을 사용하여 구현하는 이유는 OCP 때문이다.

> OCP(**O**pen-**C**losed **P**rinciple)
>
> 확장에 대해서는 개방적이고 수정에 대해서는 폐쇄적이어야 한다는 원칙.
>
> 즉, 기존 코드를 수정하지 않으면서 기능을 추가(확장)할 수 있어야 한다는 원칙이다.



옷을 생산하는 공장을 예로 들어 보자

옷 객체에 대응하는 Clothes 클래스가 있고 Clothes 객체를 생성하는 ClothesFactory 클래스가 있다.

옷 클래스에는 상품번호와 브랜드, 종류 속성이 있다.

```java
public class Clothes {
    private String number;
    private String brand;
    private String type;

    public Clothes(String number, String brand) {
        this.number = number;
        this.brand = brand;
    }

    public String getNumber() {
        return number;
    }

    public String getBrand() {
        return brand;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }
}

```

```java
public class ClothesFactory {
    public Clothes createClothes(String number, String brand) {
        if (number == null || number.isBlank()) {
            throw new IllegalArgumentException("number is blank.");
        }
        if (brand == null || brand.isBlank()) {
            throw new IllegalArgumentException("number is blank.");
        }

        Clothes clothes = new Clothes(number, brand);

        if (number.startsWith("ST-")) {
            clothes.setType("shirts");
        }
        if (number.startsWith("PT-")) {
            clothes.setType("pants");
        }

        return clothes;
    }
}

```

상품 번호의 앞 문자로 옷 종류 코드를 입력하는 매우 단순한 코드이다.



이 코드의 기능 확장에 대한 문제는 두 가지이다.

1. 만약 옷 종류가 많아지면 아래처럼 if문이 계속 늘어나게 된다.

```java
public class ClothesFactory {
    public Clothes createClothes(String number, String brand) {
        if (number == null || number.isBlank()) {
            throw new IllegalArgumentException("number is blank.");
        }
        if (brand == null || brand.isBlank()) {
            throw new IllegalArgumentException("number is blank.");
        }

        Clothes clothes = new Clothes(number, brand);

        if (number.startsWith("ST-")) {
            clothes.setType("shirts");
        }
        if (number.startsWith("PT-")) {
            clothes.setType("pants");
        }
        if (number.startsWith("CT-")) {
            clothes.setType("coat");
        }
        if (number.startsWith("JN-")) {
            clothes.setType("jean");
        }
        if (number.startsWith("JP-")) {
            clothes.setType("jumper");
        }

        return clothes;
    }
}

```

2. 특정 인스턴스에 속성을 추가하려면 기존 클래스를 수정해야 한다.\
   예를들어 코트 인스턴스에 단추 수(buttons) 속성을 추가하고 싶다면 Clothes 클래스를 수정해야 한다.&#x20;

이 상황은 OCP를 위반한다. \
기능을 추가할 때 기존 코드의 수정이 불가피 하기 때문이다.



이 코드를 팩토리 메서드 패턴을 적용해보자.

1. 인스턴스 생성을 추상화한다.

예시에서는 ClothesFactory의 createClothes 메서드를 추상화한다.

공통적인 부분을 제외하고 기능이 추가될 때 변경되는 부분만 추상화하는게 좋다.

```java
public interface ClothesFactory {

    default Clothes createClothes(String number, String brand) {
        validateArguments(number, brand);
        Clothes clothes = create();
        clothes.setNumber(number);
        clothes.setNumber(brand);
        return clothes;
    }

    private static void validateArguments(String number, String brand) {
        if (number == null || number.isBlank()) {
            throw new IllegalArgumentException("number is blank.");
        }
        if (brand == null || brand.isBlank()) {
            throw new IllegalArgumentException("number is blank.");
        }
    }

    Clothes create();
}

```

2. 팩토리 인터페이스와 제품 클래스의 구현체를 생성한다.

```java
public class ShirtsFactory implements ClothesFactory {

    @Override
    public Clothes create() {
        return new Shirts();
    }
}

```

```java
public class Shirts extends Clothes{
    public Shirts() {
        super();
        this.setType("Shirts");
    }
}
```

필요하다면 특정 클래스의 인스턴스를 생성할 때 만의 작업이 추가 될 수 있다.\
위에서 말했던 Coat 팩토리와 버튼 속성을 추가해보자.

```java
public class CoatFactory implements ClothesFactory {

    @Override
    public Clothes create() {
        return new Coat(2);
    }
}
```

```java
public class Coat extends Clothes{

    private int buttons;

    public int getButtons() {
        return buttons;
    }

    public void setButtons(int buttons) {
        this.buttons = buttons;
    }

    public Coat(int buttons) {
        super();
        this.setButtons(buttons);
        this.setType("Shirts");
    }
}
```

방금 Coat를 생성하는 기능을 추가했다.\
하지만 ClothesFactory, Clothes 또는 다른 클래스에 변경이 있었는가?\
No, 다른 클래스에는 전혀 변경이 없었다. CoatFactory와 Coat 클래스만 추가되었다.

즉, 기능이 추가되면서도 기존 코드는 수정이 없는 구조가 된 것이다. \[OCP]

팩토리 메서드 패턴은 이런 식의 구현 방식이다.



추가적으로 팩토리 메서드를 사용하는 클라이언트에서는 의존성 주입 등을 통해서 인터페이스 기반으로 코딩되는게 좋다.

```java
public class Client {
    private ClothesFactory clothesFactory;
    private ClothesRepository clothesRepository;

    public Client(ClothesFactory clothesFactory, ClothesRepository clothesRepository) {
        this.clothesFactory = clothesFactory;
        this.clothesRepository = clothesRepository;
    }

    public void saveClothes(String number, String name) {
        Clothes clothes = clothesFactory.createClothes(number, name);
        clothesRepository.save(clothes);
    }
}

```

그렇지 않으면 클라이언트 코드는 기능이 추가될 때 마다 수정이 필요하다. (CoatFactory 등 구현체를 사용해야 하므로)



팩토리 메서드 패턴의 장점

* OCP
* 기존 코드를 수정하지 않으면서 기능을 확장할 수 있으므로 코드가 간결해지고 유연한게 장점.

팩토리 메서드 패턴의 단점

* 클래스가 많아짐



