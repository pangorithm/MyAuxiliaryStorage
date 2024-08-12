### 빌더 패턴

복잡한 객체의 생성과 표현을 분리하여, 동일한 생성 절차에서 서로 다른 표현결과를 만들 수 있게 하는 생성 디자인 패턴. 복잡한 객체를 단계별로 구축할 수 있도록 하며, 최종적으로 구축 과정을 통해 객체를 반환단다.

```
class Car {
  public make: string;
  public model: string;
  public year: number;
  public options: string[];

  constructor(make: string, model: string, year: number, options: string[]) {
    this.make = make;
    this.model = model;
    this.year = year;
    this.options = options;
  }
}

class CarBuilder {
  private make: string = '';
  private model: string = '';
  private year: number = 0;
  private options: string[] = [];

  setMake(make: string): CarBuilder {
    this.make = make;
    return this;
  }

  setModel(model: string): CarBuilder {
    this.model = model;
    return this;
  }

  setYear(year: number): CarBuilder {
    this.year = year;
    return this;
  }

  addOption(option: string): CarBuilder {
    this.options.push(option);
    return this;
  }

  build(): Car {
    return new Car(this.make, this.model, this.year, this.options);
  }
}

// 사용 예:
const car: Car = new CarBuilder()
  .setMake('Tesla')
  .setModel('Model X')
  .setYear(2021)
  .addOption('Autopilot')
  .addOption('Long Range Battery')
  .build();

console.log(car);
```

### 장점

생성자 함수를 사용할 경우 각 인수의 종류와 순서를 지켜줘야 하지만, 빌더 패턴을 사용할 경우 한번에 모든 인수를 전달하지 않고 단계별로 나누어서 생성할 수 있다.(이때 체이닝 기법을 통해 각 단계를 매끄럽게 이을 수도 있다.) 이로 인해 동일한 생성 과정에서도 다양한 표현을 생성할 수 있는 유연성을 제공한다.

### 단점

별도의 Builder 클래스가 필요하므로 객체의 생성 과정을 추상화하고 분리하는데 시간이 소요되며, 복잡성과 코드 양이 증가 한다. 또한 최종 객체 생성을 위해 임시 객체가 생성되어 메모리를 차지하게 된다.

### 사용 예시

-   **SQL Query Builder**
-   **Document Builder**
-   **Test Data Builder**
-   **GUI Builder**

### SQL Query Builder 라이브러리 knex.js의 select문 예시 코드

```
knex('users')
  .join('orders', 'users.id', '=', 'orders.user_id') // 'users'와 'orders' 테이블 JOIN
  .select('users.id', 'users.name', knex.raw('COUNT(orders.id) as total_orders')) // 사용자 ID, 이름, 총 주문 수
  .where('orders.order_date', '>=', '2021-01-01') // 2021년 이후 주문한 주문만 포함
  .andWhere('orders.order_date', '<=', '2021-12-31')
  .groupBy('users.id') // 사용자별로 그룹화
  .orderBy('total_orders', 'desc') // 총 주문 수 기준 내림차순 정렬
  .limit(5) // 상위 5명만 선택
  .then(topUsers => {
    // 상위 사용자의 ID 배열 생성
    const userIds = topUsers.map(user => user.id);
    
    // 상위 사용자의 최근 주문 날짜 조회를 위한 쿼리
    return knex('orders')
      .select('user_id', knex.raw('MAX(order_date) as latest_order_date'))
      .whereIn('user_id', userIds) // 상위 사용자 ID에 해당하는 주문만 선택
      .groupBy('user_id') // 사용자 ID별로 그룹화
      .then(latestOrders => {
        // 상위 사용자 정보와 그들의 최근 주문 날짜 결합
        const result = topUsers.map(user => {
          const latestOrder = latestOrders.find(order => order.user_id === user.id);
          return {
            ...user,
            latest_order_date: latestOrder ? latestOrder.latest_order_date : null
          };
        });
        console.log(result);
      });
  })
  .catch(err => console.error(err));
```