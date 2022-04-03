---
layout: pattern
title: Domain Model
folder: domain-model
permalink: /patterns/domain-model/
categories: Architectural
language: en
tags:
 - Domain
---

## Intent

> 领域模型模式提供了一种处理复杂逻辑的面向对象的方式。不是有一个过程来处理用户操作的所有业务逻辑，而是有多个对象，每个对象都处理与其相关的领域逻辑片。

## Explanation

Real world example

> 假设我们需要构建一个电子商务 Web 应用程序。在分析需求时，您会注意到您重复谈论的名词很少。这是您的客户，也是客户寻找的产品。这两个是特定于域的类，每个类都将包含一些特定于其域的业务逻辑。

In plain words

> 域模型是结合了行为和数据的域的对象模型。

Programmatic Example

在电子商务应用程序的示例中，我们需要处理客户想要购买产品并根据需要退货的领域逻辑。
我们可以使用域模型模式并创建类“客户”和“产品”，其中该类的每个实例都包含行为和数据，
并且仅表示基础表中的一条记录。这里是 `Product` 域类，包含字段 `name`、`price`、`expirationDate`，
每个产品都是特定的，`productDao` 用于使用 DB，`save` 方法用于保存产品，`getSalePrice` 方法返回此产品的折扣价。

```java
@Slf4j
@Getter
@Setter
@Builder
@AllArgsConstructor
public class Product {

    private static final int DAYS_UNTIL_EXPIRATION_WHEN_DISCOUNT_ACTIVE = 4;
    private static final double DISCOUNT_RATE = 0.2;

    @NonNull private final ProductDao productDao;
    @NonNull private String name;
    @NonNull private Money price;
    @NonNull private LocalDate expirationDate;

    /**
     * Save product or update if product already exist.
     */
    public void save() {
        try {
            Optional<Product> product = productDao.findByName(name);
            if (product.isPresent()) {
                productDao.update(this);
            } else {
                productDao.save(this);
            }
        } catch (SQLException ex) {
            LOGGER.error(ex.getMessage());
        }
    }

    /**
     * Calculate sale price of product with discount.
     */
    public Money getSalePrice() {
        return price.minus(calculateDiscount());
    }

    private Money calculateDiscount() {
        if (ChronoUnit.DAYS.between(LocalDate.now(), expirationDate)
                < DAYS_UNTIL_EXPIRATION_WHEN_DISCOUNT_ACTIVE) {

            return price.multipliedBy(DISCOUNT_RATE, RoundingMode.DOWN);
        }

        return Money.zero(USD);
    }
}
```

这里是 `Customer` 域类，其中包含字段 `name`、`money` 字段是特定于每个客户的、`customerDao` 用于与 DB 合作、`save` 用于保存客户、
`buyProduct` 用于将产品添加到购买和退出money，从购买中删除产品并退货的`returnProduct`，
用于打印客户购买和货币余额的`showPurchases`和`showBalance`方法。

```java
@Slf4j
@Getter
@Setter
@Builder
public class Customer {

    @NonNull private final CustomerDao customerDao;
    @Builder.Default private List<Product> purchases = new ArrayList<>();
    @NonNull private String name;
    @NonNull private Money money;

    /**
     * Save customer or update if customer already exist.
     */
    public void save() {
        try {
            Optional<Customer> customer = customerDao.findByName(name);
            if (customer.isPresent()) {
                customerDao.update(this);
            } else {
                customerDao.save(this);
            }
        } catch (SQLException ex) {
            LOGGER.error(ex.getMessage());
        }
    }

    /**
     * Add product to purchases, save to db and withdraw money.
     *
     * @param product to buy.
     */
    public void buyProduct(Product product) {
        LOGGER.info(
                String.format(
                        "%s want to buy %s($%.2f)...",
                        name, product.getName(), product.getSalePrice().getAmount()));
        try {
            withdraw(product.getSalePrice());
        } catch (IllegalArgumentException ex) {
            LOGGER.error(ex.getMessage());
            return;
        }
        try {
            customerDao.addProduct(product, this);
            purchases.add(product);
            LOGGER.info(String.format("%s bought %s!", name, product.getName()));
        } catch (SQLException exception) {
            receiveMoney(product.getSalePrice());
            LOGGER.error(exception.getMessage());
        }
    }

    /**
     * Remove product from purchases, delete from db and return money.
     *
     * @param product to return.
     */
    public void returnProduct(Product product) {
        LOGGER.info(
                String.format(
                        "%s want to return %s($%.2f)...",
                        name, product.getName(), product.getSalePrice().getAmount()));
        if (purchases.contains(product)) {
            try {
                customerDao.deleteProduct(product, this);
                purchases.remove(product);
                receiveMoney(product.getSalePrice());
                LOGGER.info(String.format("%s returned %s!", name, product.getName()));
            } catch (SQLException ex) {
                LOGGER.error(ex.getMessage());
            }
        } else {
            LOGGER.error(String.format("%s didn't buy %s...", name, product.getName()));
        }
    }

    /**
     * Print customer's purchases.
     */
    public void showPurchases() {
        Optional<String> purchasesToShow =
                purchases.stream()
                        .map(p -> p.getName() + " - $" + p.getSalePrice().getAmount())
                        .reduce((p1, p2) -> p1 + ", " + p2);

        if (purchasesToShow.isPresent()) {
            LOGGER.info(name + " bought: " + purchasesToShow.get());
        } else {
            LOGGER.info(name + " didn't bought anything");
        }
    }

    /**
     * Print customer's money balance.
     */
    public void showBalance() {
        LOGGER.info(name + " balance: " + money);
    }

    private void withdraw(Money amount) throws IllegalArgumentException {
        if (money.compareTo(amount) < 0) {
            throw new IllegalArgumentException("Not enough money!");
        }
        money = money.minus(amount);
    }

    private void receiveMoney(Money amount) {
        money = money.plus(amount);
    }
}
```

在“App”类中，我们创建了代表客户 Tom 的 Customer 类的新实例，并处理该客户的数据和操作，并创建 Tom 想要购买的三种产品。


```java
// Create data source and create the customers, products and purchases tables
final var dataSource = createDataSource();
deleteSchema(dataSource);
createSchema(dataSource);

// create customer
var customerDao = new CustomerDaoImpl(dataSource);

var tom =
    Customer.builder()
        .name("Tom")
        .money(Money.of(USD, 30))
        .customerDao(customerDao)
        .build();

tom.save();

// create products
var productDao = new ProductDaoImpl(dataSource);

var eggs =
    Product.builder()
        .name("Eggs")
        .price(Money.of(USD, 10.0))
        .expirationDate(LocalDate.now().plusDays(7))
        .productDao(productDao)
        .build();

var butter =
    Product.builder()
        .name("Butter")
        .price(Money.of(USD, 20.00))
        .expirationDate(LocalDate.now().plusDays(9))
        .productDao(productDao)
        .build();

var cheese =
    Product.builder()
        .name("Cheese")
        .price(Money.of(USD, 25.0))
        .expirationDate(LocalDate.now().plusDays(2))
        .productDao(productDao)
        .build();

eggs.save();
butter.save();
cheese.save();

// show money balance of customer after each purchase
tom.showBalance();
tom.showPurchases();

// buy eggs
tom.buyProduct(eggs);
tom.showBalance();

// buy butter
tom.buyProduct(butter);
tom.showBalance();

// trying to buy cheese, but receive a refusal
// because he didn't have enough money
tom.buyProduct(cheese);
tom.showBalance();

// return butter and get money back
tom.returnProduct(butter);
tom.showBalance();

// Tom can buy cheese now because he has enough money
// and there is a discount on cheese because it expires in 2 days
tom.buyProduct(cheese);

tom.save();

// show money balance and purchases after shopping
tom.showBalance();
tom.showPurchases();
```

The program output:

```java
17:52:28.690 [main] INFO com.iluwatar.domainmodel.Customer - Tom balance: USD 30.00
17:52:28.695 [main] INFO com.iluwatar.domainmodel.Customer - Tom didn't bought anything
17:52:28.699 [main] INFO com.iluwatar.domainmodel.Customer - Tom want to buy Eggs($10.00)...
17:52:28.705 [main] INFO com.iluwatar.domainmodel.Customer - Tom bought Eggs!
17:52:28.705 [main] INFO com.iluwatar.domainmodel.Customer - Tom balance: USD 20.00
17:52:28.705 [main] INFO com.iluwatar.domainmodel.Customer - Tom want to buy Butter($20.00)...
17:52:28.712 [main] INFO com.iluwatar.domainmodel.Customer - Tom bought Butter!
17:52:28.712 [main] INFO com.iluwatar.domainmodel.Customer - Tom balance: USD 0.00
17:52:28.712 [main] INFO com.iluwatar.domainmodel.Customer - Tom want to buy Cheese($20.00)...
17:52:28.712 [main] ERROR com.iluwatar.domainmodel.Customer - Not enough money!
17:52:28.712 [main] INFO com.iluwatar.domainmodel.Customer - Tom balance: USD 0.00
17:52:28.712 [main] INFO com.iluwatar.domainmodel.Customer - Tom want to return Butter($20.00)...
17:52:28.721 [main] INFO com.iluwatar.domainmodel.Customer - Tom returned Butter!
17:52:28.721 [main] INFO com.iluwatar.domainmodel.Customer - Tom balance: USD 20.00
17:52:28.721 [main] INFO com.iluwatar.domainmodel.Customer - Tom want to buy Cheese($20.00)...
17:52:28.726 [main] INFO com.iluwatar.domainmodel.Customer - Tom bought Cheese!
17:52:28.737 [main] INFO com.iluwatar.domainmodel.Customer - Tom balance: USD 0.00
17:52:28.738 [main] INFO com.iluwatar.domainmodel.Customer - Tom bought: Eggs - $10.00, Cheese - $20.00
```

## Class diagram

![](./etc/domain-model.urm.png "domain model")

## Applicability

当您的域逻辑很复杂并且复杂性会迅速增长时，请使用域模型模式，因为这种模式可以很好地处理不断增加的复杂性。
否则，它是组织领域逻辑的更复杂的解决方案，因此不应将领域模型模式用于具有简单领域逻辑的系统，因为理解它的成本和数据源的复杂性超过了这种模式的好处。

## Related patterns

- [Transaction Script](https://java-design-patterns.com/patterns/transaction-script/)

- [Table Module](https://java-design-patterns.com/patterns/table-module/)
  
- [Service Layer](https://java-design-patterns.com/patterns/service-layer/)

## Credits

* [Domain Model Pattern](https://martinfowler.com/eaaCatalog/domainModel.html)
* [Patterns of Enterprise Application Architecture](https://www.amazon.com/gp/product/0321127420/ref=as_li_qf_asin_il_tl?ie=UTF8&tag=javadesignpat-20&creative=9325&linkCode=as2&creativeASIN=0321127420&linkId=18acc13ba60d66690009505577c45c04)
* [Architecture patterns: domain model and friends](https://inviqa.com/blog/architecture-patterns-domain-model-and-friends)
