CQRS framework in go
=======
[![GoDoc](https://godoc.org/github.com/andrewwebber/cqrs?status.svg)](https://godoc.org/github.com/andrewwebber/cqrs)
[![Build Status](https://drone.io/github.com/andrewwebber/cqrs/status.png?foo=bar)](https://drone.io/github.com/andrewwebber/cqrs/latest)

[Example test scenario (inmemory)](https://github.com/andrewwebber/cqrs/blob/master/cqrs_test.go)
[Example test scenario (couchbase, rabbitmq)](https://github.com/andrewwebber/cqrs/blob/master/infrastructureexample/example_test.go)

## Example Bank Account Scenario
The example test scenario is of a simple bank account that seeks to track, using event sourcing, a
customers balance and login password

The are two make areas of concern at the application level, the **Write model** and **Read model**.
The read model is aimed to facilitate fast reads (read model projections)
The write model is where the business logic get executed and asynchronously notifies the read models

## Write model - Using Event Sourcing
### Account
```go
type Account struct {
  cqrs.EventSourceBased

  FirstName    string
  LastName     string
  EmailAddress string
  PasswordHash []byte
  Balance      float64
}
```
To compensate for golang's lack of inheritance, a combination of type embedding and a call convention
pattern are utilized

```go
func NewAccount(firstName string, lastName string, emailAddress string, passwordHash []byte, initialBalance float64) *Account {
  account := new(Account)
  account.EventSourceBased = cqrs.NewEventSourceBased(account)

  event := AccountCreatedEvent{firstName, lastName, emailAddress, passwordHash, initialBalance}
  account.Update(event)
  return account
}
```

### Account Events
```go
type AccountCreatedEvent struct {
  FirstName      string
  LastName       string
  EmailAddress   string
  PasswordHash   []byte
  InitialBalance float64
}

type EmailAddressChangedEvent struct {
  PreviousEmailAddress string
  NewEmailAddress      string
}

type PasswordChangedEvent struct {
  NewPasswordHash []byte
}

type AccountCreditedEvent struct {
  Amount float64
}

type AccountDebitedEvent struct {
  Amount float64
}
```
Events souring events are raised using the embedded **Update** function. These events will also be published
to the read models
```go
func (account *Account) ChangePassword(newPassword string) error {
  if len(newPassword) < 1 {
    return errors.New("Invalid newPassword length")
  }

  hashedPassword, err := GetHashForPassword(newPassword)
  if err != nil {
    panic(err)
  }

  account.Update(PasswordChangedEvent{hashedPassword})

  return nil
}
```

## Read Model
### Accounts projection
```go
type ReadModelAccounts struct {
  Accounts map[string]*AccountReadModel
}

type AccountReadModel struct {
  ID           string
  FirstName    string
  LastName     string
  EmailAddress string
  Balance      float64
}
```
### Users projection
```go
type UsersModel struct {
  Users    map[string]*User
}

type User struct {
  ID           string
  FirstName    string
  LastName     string
  EmailAddress string
  PasswordHash []byte
}
```

## Infrastructure
There are a number of key elements to the CQRS infrastructure.
- Event sourcing repository (a repository for event sourcing based business objects)
- Event publisher (publishes new events to an event bus)
- Event handler (dispatches received events to call handlers)
- Command publisher (publishes new commands to a command bus)
- Command handler (dispatches received commands to call handlers)

### Event sourcing and integration events
Nested packages within this repository show example implementations using Couchbase Server and RabbitMQ.
The core library includes in-memory implementations for testing and quick prototyping
```go
persistance := cqrs.NewInMemoryEventStreamRepository()
bus := cqrs.NewInMemoryEventBus()
repository := cqrs.NewRepositoryWithPublisher(persistance, bus)
```

With the infrastructure implementations instantiated a stock event dispatcher is provided to route received
events to call handlers
```go
readModel := NewReadModelAccounts()
usersModel := NewUsersModel()

eventDispatcher := cqrs.NewVersionedEventDispatchManager(bus)
eventDispatcher.RegisterEventHandler(AccountCreatedEvent{}, func(event cqrs.VersionedEvent) error {
  readModel.UpdateViewModel([]cqrs.VersionedEvent{event})
  usersModel.UpdateViewModel([]cqrs.VersionedEvent{event})
  return nil
})
```

We can also register a **global** handler to be called for all events.
This becomes useful when logging system wide events and when our read models are smart enough to filter out irrelevant events
```go
integrationEventsLog := cqrs.NewInMemoryEventStreamRepository()
eventDispatcher.RegisterGlobalHandler(func(event cqrs.VersionedEvent) error {
  integrationEventsLog.SaveIntegrationEvent(event)
  readModel.UpdateViewModel([]cqrs.VersionedEvent{event})
  usersModel.UpdateViewModel([]cqrs.VersionedEvent{event})
  return nil
})
```

Within your read models the idea is that you implement the updating of your pre-pared read model based upon the
incoming event notifications

### Commands

Commands are processed by command handlers similar to event handlers.
We can make direct changes to our write model and indirect changes to our read models by correctly processing commands and then raising integration events upon command completion.

```go
commandBus := cqrs.NewInMemoryCommandBus()
commandDispatcher := cqrs.NewCommandDispatchManager(commandBus)
RegisterCommandHandlers(commandDispatcher, repository)
```

Commands can be issued using a command bus. Typically a command is a simple struct.
The application layer command struct is then wrapped within a cqrs.Command using the cqrs.CreateCommand helper function

```go
changePasswordCommand := cqrs.CreateCommand(
  ChangePasswordCommand{accountID, "$ThisIsANOTHERPassword"})
commandBus.PublishCommands([]cqrs.Command{changePasswordCommand})
```

The corresponding command handler for the **ChangePassword** command plays the role of a DDD aggregate root; responsible for the consistency and lifetime of aggregates and entities within the system)
```go
commandDispatcher.RegisterCommandHandler(ChangePasswordCommand{}, func(command cqrs.Command) error {
  changePasswordCommand := command.Body.(ChangePasswordCommand)
  // Load account from storage
  account, err := NewAccountFromHistory(changePasswordCommand.AccountID, repository)
  if err != nil {
    return err
  }

  account.ChangePassword(changePasswordCommand.NewPassword)

  // Persist new events
  repository.Save(account)  
  return nil
})
```

As the read models become consistant, within the tests, we check at the end of the test if everything is in sync
```go
if account.EmailAddress != lastEmailAddress {
  t.Fatal("Expected emailaddress to be ", lastEmailAddress)
}

if account.Balance != readModel.Accounts[accountID].Balance {
  t.Fatal("Expected readmodel to be synced with write model")
}
```
