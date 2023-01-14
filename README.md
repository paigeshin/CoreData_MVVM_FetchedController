# FechtedController

### CoreDataManager

```swift
import Foundation
import CoreData

class CoreDataManager {
    
    let persistentStoreContainer: NSPersistentContainer
    static let shared = CoreDataManager()
    
    private init() {
        persistentStoreContainer = NSPersistentContainer(name: "BudgetAppModel")
        persistentStoreContainer.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Unable to initialize Core Data \(error)")
            }
        }
    }
    
}
```

### BaseModel

```swift
import Foundation
import CoreData

protocol BaseModel {
    static var viewContext: NSManagedObjectContext { get }
    func save() throws
    func delete() throws
}

extension BaseModel where Self: NSManagedObject {
    
    static var viewContext: NSManagedObjectContext {
        CoreDataManager.shared.persistentStoreContainer.viewContext
    }
    
    func save() throws {
        try Self.viewContext.save()
    }
    
    func delete() throws {
        Self.viewContext.delete(self)
        try save()
    }
    
}
```

### Budget + Extension

```swift
import Foundation
import CoreData

extension Budget: BaseModel {
    
    static var all: NSFetchRequest<Budget> {
        let request = Budget.fetchRequest()
        request.sortDescriptors = []
        return request 
    }
    
}
```

### BudgetListViewModel

```swift
import Foundation
import CoreData

@MainActor
class BudgetListViewModel: NSObject, ObservableObject {
    
    @Published var budgets = [BudgetViewModel]()
    private let fetchedResultsController: NSFetchedResultsController<Budget>
    private (set) var context: NSManagedObjectContext
    init(context: NSManagedObjectContext) {
        
        self.context = context
        fetchedResultsController = NSFetchedResultsController(fetchRequest: Budget.all, managedObjectContext: context, sectionNameKeyPath: nil, cacheName: nil)
        super.init()
        fetchedResultsController.delegate = self
        
        do {
            try fetchedResultsController.performFetch()
            guard let budgets = fetchedResultsController.fetchedObjects else {
                return
            }
            
            self.budgets = budgets.map(BudgetViewModel.init)
        } catch {
            print(error)
        }
        
    }
    
    func deleteBudget(budgetId: NSManagedObjectID) {
        do {
            guard let budget = try context.existingObject(with: budgetId) as? Budget else {
                return
            }
            
            try budget.delete()
            
        } catch {
            print(error)
        }
    }
    
    
}

extension BudgetListViewModel: NSFetchedResultsControllerDelegate {
    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        
        guard let budgets = controller.fetchedObjects as? [Budget] else {
            return
        }
        
        self.budgets = budgets.map(BudgetViewModel.init)
    }
}

struct BudgetViewModel: Identifiable {
    
    private var budget: Budget
    
    init(budget: Budget) {
        self.budget = budget
    }
    
    var id: NSManagedObjectID {
        budget.objectID
    }
    
    var title: String {
        budget.title ?? ""
    }
    
    var total: Double {
        budget.total
    }
    
}
```

### AddBudgetViewModel

```swift
import Foundation
import CoreData

class AddBudgetViewModel: ObservableObject {
    
    @Published var name: String = ""
    @Published var total: String = ""
    
    var context: NSManagedObjectContext
    
    init(context: NSManagedObjectContext) {
        self.context = context
    }
    
    func save() {
        
        do {
            let budget = Budget(context: context)
            budget.title = name
            budget.total = Double(total) ?? 0
            try budget.save()
        } catch {
            print(error)
        }
        
    }
    
}
```

### App

```swift
import SwiftUI

@main
struct BudgetAppApp: App {
    var body: some Scene {
        WindowGroup {
            
            let viewContext = CoreDataManager.shared.persistentStoreContainer.viewContext
            ContentView(vm: BudgetListViewModel(context: viewContext))
                .environment(\.managedObjectContext, viewContext)
            
        }
    }
}
```
# CoreData_MVVM_FetchedController
