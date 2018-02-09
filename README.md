

## Description

This npm package provides an abstract transaction implementation on a node server.

___Example of simple transaction___

The defined transaction will execute the code block and return a promise with its state

```javascript
const {startTransaction} = require('abstract-transaction');

startTransaction()
    .then((transaction) => service.update(transaction,args))
    .then((result) => console.info('it is committed'))
    .catch((error) => console.info('it is rolled back'))
```


___Example of inner transaction rollback___

When a transaction is defined within another transaction, the parent transaction can handle inner transaction failures.
By default if an inner transaction rolls back, the main will rollback.

but if the promise of a rolled back inner transaction resolves. Only the inner transaction will rollback, not the parent transaction.

Here we force the rollback by call calling rollback().
If the transactional code throws an error, this will also trigger the rollback.


```javascript
const {startTransaction} = require('abstract-transaction');

const promise = startTransaction()
    .then((transaction) => runInnerTransaction(transaction))
    .catch(() => console.info('it is rolled back'))


function runInnerTransaction(parentTransaction)
    return parentTransaction.startInner()
        .then((transaction) => transaction.rollback()))
        .catch((err) => {
            console.info('it is rolled back')
            // the parent transaction will roll back only if it is rejected or thrown.
            throw err;
        );
}
```

___Example when a transaction with inner transaction rolls back___

When a transaction is defined within another transaction, the parent transaction can handle inner transaction failures.

```javascript
const {startTransaction} = require('abstract-transaction');

const promise = startTransaction({
      onCommit: () => { 
          console.info('This is executed after main transaction has committed');
      }
 })
.then((transaction) => {
    runInnerTransaction(transaction));
    transaction.rollback();
})
.catch(() => console.info('it is rolled back'))


function runInnerTransaction(parentTransaction)
    return parentTransaction.startInner({
        onCommit: (result) => { 
            console.info('This message will never show, since the main transaction has rolled back');
        }
        onRollback: () => { 
            console.info('This also is executed after main transaction has rolled back');
        }
    })
    .then(() => service.update(transaction,args))
    .then(() => console.info('it is partially committed, but final commit only happens if the main transaction commits'))
    .catch(() => console.info('it is not executed, since the main transaction failed, not the inner one'))
}
```

## Defining the implementation 
The default implementation has no effect. The commit and rollback code needs implementing.
An implementation class must be provided before using transaction

Example of setting the default implementation class

```javascript
const {setTransactionImplementationClass, getCoreTransactionClass} = require('abstract-transaction');

class TransactionImplementation extends getCoreTransactionClass() {
    constructor(parentTransaction, options) {
        super(parentTransaction, {
            processBegin: _.noop,
            processCommit: _.noop,
            processRollback: _.noop,
            processInnerBegin: _.noop,
            processInnerCommit: _.noop,
            processInnerRollback: _.noop
        },
        options);
    }
}

setTransactionImplementationClass(TransactionImplementation);
```


## API

___setTransactionImplementationClass function___

etTransactionImplementationClass(class)

class is the class to implement the begin, commit and rollback.


___startTransaction function___

const transaction = startTransaction(options)

Options: 

- onCommit callback is executed after the transaction commits.

- onRollback callback is executed after the transaction rollbacks.

- implementationClass is the implementation class that will be used instead of the configured class


___reUseOrCreateTransaction function___

const transaction = reUseOrCreateTransaction(, parentTransaction, options)

Options: 

onCommit callback is executed after the transaction commits.

onRollback callback is executed after the transaction rollbacks.

implementationClass is the implementation class that will be used instead of the configured class


___defineTransaction function___

Lower level function

const transaction = defineTransaction(requirements, parentTransaction, options)

Requirements : USE, NEW, REUSE_OR_NEW

Options: 

onCommit callback is executed after the transaction commits.

onRollback callback is executed after the transaction rollbacks.

implementationClass is the implementation class that will be used instead of the configured class


___Transaction object methods___

then((transaction) => ...) returns a promise and receives the transaction code to execute

execute((transaction) => ...) returns a promise. This is the same as then function

startInner(options) returns an inner transaction

rollback(), rollback the transaction

