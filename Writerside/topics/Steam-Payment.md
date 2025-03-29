# Steam payments (Microtransactions)

## Call flow (browser client)
```mermaid
sequenceDiagram
participant Client
participant Catena
participant Steam

Client->>Catena: Request to purchase item(s)
Catena->>Steam: InitTxn()
Steam->>Catena: Returns transaction ID

alt Using browser/web request
Catena->>Client: Steam URL to authorize purchase
Client->>Catena: Callback when purchase is authorized/rejected
else In game with overlay
Steam->>Client: Notification in overlay
Client->>Catena: Callback when purchase is authorized/rejected
end

loop
Catena->>Steam: QueryTxn()
Steam->>Catena: Current transaction state

alt Transaction authorized
Catena->>Steam: FinalizeTxn()
Steam->>Catena: Success
Catena->>Client: Item(s) granted
else Transaction rejected or transaction abandoned
Catena->>Client: Item(s) not granted
end
end
```

## Steam order/transaction status
```mermaid
flowchart

Created -- Issue InitTxn --> Init
Init -- Rejected by client --> Failed
Init -- Approved by client --> Approved
Approved -- Finalized by Catena --> Succeeded
Init -- Timed out by Catena --> Abandoned
Created --> Invalid
Succeeded --> Verified
Verified --> Verified
```

`Created` is the state used by Catena to record/journal that a transaction is about to take place Catena; it is not a state in Steam.

`Abandoned` is a state reached by Catena when a transaction has not been approved/rejected by a deadline; it is not a state in Steam.

`Invalid` is a state reached by Catena when a transaction is recorded but an order ID collision prevents it from being started; it is not a state in Steam.

`Verified` is a state reached by Catena after a transaction has succeeded, each time it is verified.

`Reversed` is a state reached by Catena after a transaction when a transaction has been refunded/charged back.