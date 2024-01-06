# Architectural-Best-Practices-in-Action

![CleanArchitecture copy](https://github.com/Vinodh-G/Architectural-Best-Practices-in-Action/assets/5305527/2f40485d-7d7e-4495-b9d8-a16cf890b5b5)

Welcome to "Mobile: Architectural Best Practices in Action" - a series where we embark on a journey through the intricate world of mobile application development. In this series, we won't just talk theory; we'll roll up our sleeves and dive into real-world scenarios, use cases, and challenges that developers often face.

### What to Expect:
In each installment, we'll pick a specific situation, a scenario, or a use case that reflects the complex landscape of mobile app development. But we won't stop at merely identifying the challenge - we'll approach it with an architectural mindset, dissecting the problem and crafting a robust solution.
### Why Architectural Mindset Matters:
Why do we bother with an architectural mindset? Because in the dynamic and fast-paced world of mobile development, it's not just about getting the app to work; it's about crafting a solution that stands the test of time, scales gracefully, and remains maintainable as your app evolves.
### Diving into the Design:
We'll talk about design patterns, styles, and best practices. But here's the deal - we won't always take the easy road. We're going to chat about why we choose one design over another, even if it seems a bit tougher. It's about making smart choices for great results.
### Join the Conversation:
This series isn't just about me sharing insights; it's a conversation. I invite you to engage, ask questions, and share your own experiences. Let's learn together as we resolve the complexity of mobile architectural design.
With that let us get into our first use case.

## User Story 1
We need to develop new UI components for the app, which will be part of the existing User Profile Screen.
Account User Orders UI: Lists all recent user orders.
Account User Address UI: Lists all user addresses.
Now, there’s a AccountService class extensively used in the mobile app. It's a legacy class, developed a while ago, singleton, and internally handles authentication, token exchange, and fetches data locally in case of offline mode. Here's a simplified version of the class:

```
final class AccountService {
    
    static let shared: AccountService = AccountService()
    private init () {}
 
    func fetchData(requestParam: UserRequestParam) async -> (Data?, AccountServiceError?) {
        // The internal logic for fetching the data for the url
        // using network classes like urlsession, datatask
        return (Data(), nil)
    }
}
```

### Approach
let’s break down and analyze the given requirement
**Objective**: Develop new UI components for the app to enhance the existing User Profile Screen.
Components to Develop:
**Account User Orders** ➔ UI Display a list of all recent user orders.
**Account User Address UI** ➔ Display a list of all user addresses.

### Existing Infrastructure:
There’s an AccountService class already in use within the mobile app.
It’s a legacy class, developed some time ago.
The class is implemented as a singleton.
It handles crucial functionalities:
Authentication & Token exchange
Local data fetching (for offline mode)


### Key Considerations:
**Legacy Status**: Since AccountService is acknowledged as a legacy class, indicating that it can be or soon a later replaced.
**Singleton Pattern**: The class is implemented as a singleton. Singletons, easy to use, but it gives us challenges such as tight coupling and reduced testability.
**Functionality**: The fetchData function is asynchronous, suggesting that it likely performs network operations. The AccountService handles local data fetching, indicating offline support.
**Scope of UI Components**:The new UI components are meant to be integrated into the existing User Profile Screen.


Now, let’s create UserOrderListView and UserAddressListView – the two new UI components that will use the existing legacy service, AccountService.
Not a big fan of singleton, feel it is an anti pattern. Singletons can be ok. But if you are not careful, they aren’t just an anti-pattern.
Also for our feature and timeline, we don’t have bandwidth to develop new modular AccountService from scratch, we have to use the existing one.


An easy and tempting approach is to start the implementation by using AccountService.shared inside UserOrderListView or, if we are following MVVM, inside UserOrderListViewModel.

```
class UserOrderListViewModel {
    
    func fetchOrders(userId: String) async -> [OrderViewData] {
        let (orders, error) = await AccountService.shared.fetchData(requestParam: UserRequestParam(orderDetails))
        // Here the logic of converting data to type
        // Handling error
        return []
    }
}
```
```
class UserAddressListViewModel {
    
    func fetchAddress(userId: String) async -> [AddressViewData] {
        let (address, error) = await AccountService.shared.fetchData(requestParam: UserRequestParam(addresses))
        // Here the logic of converting data to type
        // Handling error
        return []
    }
}
```
While this approach works, it comes with disadvantages:
**Tight Coupling**: Yea it looks like we have coupled the AccountService directly to the view models, since it was readily available, but we know the AccountService is legacy class, it will get deprecated after sometime, depends how much refactoring fund is available :P, what if we have new AccountService which is not singleton later stage, then we have get into these view models and make changes, cause low confidence on product, which will again cost development and testing.

**Unit testing**: Say if we have to unit test the view models, which might reach to the network for fetching the user related information, which is time consuming, not required to unit test behaviour of the viewModel, then if we have to mock the AccountService singleton, which was kind not that hectic on Objective C, but in swift it lot difficult or we have to use 3rd party lib for mocking singleton in swift.
**Modularity**: What if we are developing the new screens (`UserOrderListView` and `UserAddressListView`) in new module/framework, then the new framework has the direct dependency on the existing module which contains AccountService, which we dont want it, because the purpose of new module is ti keep it independant and easily shippable to other apps, if we have direct dependecy on the AccountService Module, we are forced to ship this new framework along with AccountService Module.


Ok, thats a lot of issues, on this easy fix, lets try extending the 2 seperate functions for fetching orders and addresses

```
extension AccountService {
    func fetchOrder(userId: String) async -> Result<[UserOrders], UserOrdersError> {
        let (orders, error) = await AccountService.shared.fetchData(requestParam: UserRequestParam(userId))
                // Here the logic of converting data to type
                // Handling error
        return Result.success([])
    }
}
```
```
extension AccountService {
    func fetchAddresses(userId: String) async -> Result<[UserAddresses], UserAddressesError> {
        let (address, error) = await AccountService.shared.fetchData(requestParam: UserRequestParam(userId))
                // Here the logic of converting data to type
                // Handling error
        return Result.success([])
    }
}
```

Now, we have two separate functions for two different use cases, still part of the same AccountService singleton.
As we can see in Clean Architecture graph, we have different layers in the application. The main rule is not to have dependencies from inner layers to outers layers. There can only be dependencies from outer layer inward.
After grouping all layers we have: **Presentation**, **Domain** and **Data layers**.


Following a `protocol-oriented approach`, we can remove the dependency on AccountService from UserOrderListViewModel and UserAddressListViewModel, also I would remove the import and impressions related to AccountService from the UserOrderListViewModel & UserAddressListViewModel
```
class UserOrderListViewModel {
    private let orderService: FetchUserOrder
    init(orderService: FetchUserOrder) {
        self.orderService = orderService
    }

    func fetchOrders(userId: String) async -> [OrderViewData] {

    }
}
```
```
class UserAddressListViewModel {   
    private let addressService: FetchUserAddressess
    init(addressService: FetchUserAddressess) {
        self.addressService = addressService
    }

    func fetchAddress(userId: String) async -> [AddressViewData] {

    }
}
```
Thats fine, we have removed the impressions of AccountService Singleton, but we have not implemented how to get the user order list or user address. And we can see there are use case declared and injected to the respective viewmodel, FetchUserOrder use case for fetching user orders, FetchUserAddressess usecase for fetching user address.

Ok, now UserOrderListViewModel should use `FetchUserOrder` protocol interface to fetch user orders and UserAddressListViewModel should use `FetchUserAddressess` protocol interface to fetch user addresses, lets go and declare the functions on the protocols.
```
protocol FetchUserOrder {
    func fetchOrder(userId: String) async -> Result<[UserOrders], UserOrdersError>
}
```
```
protocol FetchUserAddressess {
    func fetchAddresses(userId: String) async -> Result<[UserAddresses], UserAddressesError>
}
```

Now lets use these functions inside the ViewModels to get the respective details, the view models will look like this
```
class UserOrderListViewModel {

    private let orderService: FetchUserOrder

    func fetchOrders(userId: String) async -> [OrderViewData] {
        let ordersResult = await orderService.fetchOrder(userId: "userId")
        // Here the logic of converting data to type
        // Handling error
        return []
    }
}
```
```
class UserAddressListViewModel {
    
    private let addressService: FetchUserAddressess
    init(addressService: FetchUserAddressess) {
        self.addressService = addressService
    }
    
    func fetchAddress(userId: String) async -> [AddressViewData] {
        let addressesResult = await addressService.fetchAddresses(userId: "userId")
        // Here the logic of converting data to type
        // Handling error
        return []
    }
}
```

Cool, we have things sorted from the newly created view models, we don’t have tight coupling with AccountService singleton, and we can inject the LocalMocks confirming to FetchUserOrder and FetchUserAddresses to unit test UserOrderListViewModel & UserAddressListViewModel.
But who is going to get the details, since we have no new service for user related details, have to use the existing AccountService singleton, if we scroll up we can see we have extended the singleton AccountService with 2 functions
`fetchOrder(userId: String)` and `fetchAddress(userdId: String)`

Now we can connect the `FetchUserOrder protocol to fetchOrder(userId: String)` and `FetchUserAddressess protocol to fetchAddress(userdId: String`), which means we can make the AccountService singleton to confirm to FetchUserOrder and FetchUserAddressess, since it implements both the methods.
```
extension AccountService: FetchUserOrder {
    func fetchOrder(userId: String) async -> Result<[UserOrders], UserOrdersError> {
        let (orders, error) = await AccountService.shared.fetchData(requestParam: UserRequestParam(userId))
                // Here the logic of converting data to type
                // Handling error
        return Result.success([])
    }
}
```
```
extension AccountService: FetchUserAddressess {
    func fetchAddresses(userId: String) async -> Result<[UserAddresses], UserAddressesError> {
        let (address, error) = await AccountService.shared.fetchData(requestParam: UserRequestParam(userId))
                // Here the logic of converting data to type
                // Handling error
        return Result.success([])
    }
}
```
Now this same singleton should be injected in the ViewModels when getting created, which could be taking place in the Coordinator or App Main Module.

Now the above has `solved Tight Coupling, Unit testing, Modularity problem`, as you see we `don’t` have direct association with singleton AccountService, we can `easily unit test the ViewModels injecting the mock for FetchUserOrder, FetchUserAddress`, and we can `reuse` the same UserAddressListViewModel to different app, if this is extracted out as Module, by injecting the app specific implementation.

we’ve taken a close look at the challenges presented by the existing AccountService singleton. By adopting a `protocol-oriented approach` and `embracing dependency injection`, we've untangled tight couplings, improved unit testing capabilities, and fortified modularity.

Our exploration doesn’t end here. In the next blog, we’ll dive into another real-world scenario, applying architectural best practices to solve unique challenges. As we continue our journey through mobile architecture, expect more hands-on examples, practical insights, and strategies for crafting resilient and scalable solutions.
