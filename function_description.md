Reseller transaction function

Updates for https://statystech.atlassian.net/browse/RP-300 in bold orange.
1. Conduct function refactoring;
2. Simplified calculation approaches;
3. Fixed deadlocks for edge cases;

**Function**  
resellerTransaction (  
  reseller_id=NULL,  
  transaction_type=NULL,  
  transaction_amount=NULL,  
  order_id=NULL,  
  parcel_line_id=NULL,  
  quantity=NULL,  
  warehouse_order_id=NULL,  
  order_line_id=NULL,  
  warehouse_id=NULL,  
  parcel_id=NULL);  
  
**Function arguments**

- reseller_id : int  
- transaction_type : int  
- transaction_amount : float  
- order_id : int  
- parcel_line_id : int  
- quantity : int  
- warehouse_order_id : int  
- order_line_id : int  
- warehouse_id : int  
- parcel_id : int  

**Config parameters**  
order_date = '2022-06-01';  

**Transaction types**  
701 - ‘Top-up balance’; # positive amount  
702 - ‘Top-up balance with credit’; # positive amount  
711 - ‘Charge cost of goods’; # negative amount, positive quantity  
712 - ‘Charge marketplace fee’; # negative amount, positive quantity  
721 - 'Refund cost of goods'; # positive amount, negative quantity  
722 - ‘Refund marketplace fee'; # positive amount, negative quantity  
713 - ‘Charge shipping cost’; # negative amount  
723 - ‘Refund shipping cost’; # positive amount  

**Business logic**  

1. Order date validation:  
  #Process orders only from a particular date
    - If transaction_type != 701 or 702:  
        - If order_id != NULL:
            - If orders.orderDate < order_date or orders.buyerAccountID is NULL:  
              WHERE orders.orderID = order_id:  
              - Exit from the function;  
        - Elif warehouse_order_id != NULL:  
            - If orders.orderDate < order_date or orders.buyerAccountID is NULL:  
              WHERE warehouseOrder.warehouseOrderID = warehouse_order_id  
              AND warehouseOrder.orderID = orders.orderID:
                - Exit from the function;  
        - Elif order_line_id != NULL:  
            - If orders.orderDate < order_date or orders.buyerAccountID is NULL:  
              WHERE orderLine.orderLineID = order_line_id  
              AND orderLine.warehouseOrderID = warehouseOrder.warehouseOrderID  
              AND warehouseOrder.orderID = orders.orderID:  
                - Exit from the function;  
        - Elif parcel_line_id != NULL:  
            - If orders.orderDate < order_date or orders.buyerAccountID is NULL:  
              WHERE parcelLine.parcelLineID = parcel_line_id  
              AND parcelLine.orderLineID = orderLine.orderLineID  
              AND orderLine.warehouseOrderID = warehouseOrder.warehouseOrderID  
              AND warehouseOrder.orderID = orders.orderID:
                - Exit from the function;
        - Elif parcel_id != NULL:
            - If orders.orderDate < order_date or orders.buyerAccountID is NULL:  
                WHERE parce.parcelD = parcel_id  
                AND parcel.warehouseOrderID = warehouseOrder.warehouseOrderID  
                AND warehouseOrder.orderID = orders.orderID:
                  - Exit from the function;  
2. Default:  
    Return exception;  
3. Case transaction_type = 701 and reseller_id != NULL and transaction_amount > 0:  
   _top-up resellers' account balance with own money_  
    - Create a record in resellerTransaction table where:  
        - resellerTransaction.resellerID = reseller_id;  
        - resellerTransaction.transactionTypeID = 701;  
        - resellerTransaction.transactionAmount = encrypted(transaction_amount);  
        - reseller.resellerAccountBalance += encrypted(transaction_amount);  
      _update reseller account balance_  
4. Case transaction_type = 702 and reseller_id != NULL and transaction_amount > 0:
   _top-up resellers' account balance with marketplace credit_
    - Create a record in resellerTransaction table where:
        - resellerTransaction.resellerID = reseller_id;
        - resellerTransaction.transctionTypeID = 702;
        - resellerTransaction.transactionAmount = encrypted(transaction_amount);
    - reseller.resellerAccountBalance += encrypted(transaction_amount); 
  update reseller account balance  
5. Case transaction_type = 711 and order_id != NULL:  
   _new order submission_  
    - For each warehouseOrder in order:
        - For each orderLine in warehouseOrder:
            - Create a record in resellerTransaction table where:  
              _transaction for cost of goods payment_
                  - resellerTransaction.resellerID = website.resellerID  
                    WHERE website.websiteID = buyerAccount.websiteID  
                    AND buyerAccount.buyerAccountID = orders.buyerAccountID  
                    AND orders.orderID = order_id;  
                  - resellerTransaction.warehouseID = warehouseOrder.warehouseID;  
                  - resellerTransaction.orderID = order_id;  
                  - resellerTransaction.warehouseOrderID = warehouseOrder.warehouseOrderID;  
                  - resellerTransaction.orderLineID = orderLine.orderLineID;  
                  - resellerTransaction.transactionTypeID = 711;  
                  - resellerTransaction.quantity = orderLine.quantity;  
                  - If reseller.currencyCD == warehouse.currencyCD:
                      - resellerTransaction.currencyExchangeRate = 1;  
                  - Else:  
                      - resellerTransaction.currencyExchangeRate = currencyExchangeRate.exchangeRate *                           (1+currencyExchangeRate.marketplaceMarkup)  
                        WHERE currencyExchangeRate.fromCurrencyCD = warehouse.currencyCD  
                        AND currencyExchangeRate.toCurrencyCD = reseller.currencyCD  
                        ORDER BY currencyExchangeRate.fxDate DESC  
                        LIMIT 1;  
                  - If warehouseInventory.warehouseProductPrice == NULL  
                    WHERE warehouseInventory.warehouseID = warehouseOrder.warehouseID 
                    AND warehouseInventory.productID = orderLine.productID:
                      - resellerTransaction.warehouseProductPrice = encrypted(-1);
                  - Ese:  
                      - resellerTransaction.warehouseProductPrice = encrypted(warehouseInventory.warehouseProductPrice * resellerTransaction.currencyExchangeRate);  
                  - resellerTransaction.transactionAmount = encrypted(decrypted(resellerTransaction.quantity) * decrypted(resellerTransaction.warehouseProductPrice) * -1);  
              - reseller.resellerAccountBalance = encrypted(decrypted(reseller.resellerAccountBalance) + decrypted(resellerTransaction.transactionAmount));  
                        _update reseller account balance_  
              - Create a record in resellerTransaction table where:  
                _transaction for marketplace fee_  
                  - resellerTransaction.resellerID = website.resellerID  
                    WHERE website.websiteID = buyerAccount.websiteID  
                    AND buyerAccount.buyerAccountID = orders.buyerAccountID  
                    AND orders.orderID = order_id;
                    - resellerTransaction.warehouseID = warehouseOrder.warehouseID;  
                    - resellerTransaction.orderID = order_id;  
                    - resellerTransaction.warehouseOrderID = warehouseOrder.warehouseOrderID;  
                    - resellerTransaction.orderLineID = orderLine.orderLineID;  
                    - resellerTransaction.transactionTypeID = 712; 
                    - resellerTransaction.quantity = orderLine.quantity;  
                    - If reseller.currencyCD == warehouse.currencyCD:  
                        - resellerTransaction.currencyExchangeRate = 1;  
                    - Else:  
                        - resellerTransaction.currencyExchangeRate = currencyExchangeRate.exchangeRate * (1+currencyExchangeRate.marketplaceMarkup)
                        WHERE currencyExchangeRate.fromCurrencyCD = warehouse.currencyCD
                        AND currencyExchangeRate.toCurrencyCD = reseller.currencyCD
                        ORDER BY currencyExchangeRate.fxDate DESC
                        LIMIT 1;
                    - If warehouseInventory.warehouseProductPrice == NULL  
                      WHERE warehouseInventory.warehouseID = warehouseOrder.warehouseID  
                      AND warehouseInventory.productID = orderLine.productID:  
                          - resellerTransaction.warehouseProductPrice = encrypted(-1);  
                    - Ese:  
                        - resellerTransaction.warehouseProductPrice = encrypted(warehouseInventory.warehouseProductPrice * resellerTransaction.currencyExchangeRate);  
                    - resellerTransaction.transactionAmount = encrypted(decrypted(resellerTransaction.quantity) * decrypted(resellerTransaction.warehouseProductPrice) * decrypted(reseller.resellerMarketplaceFee) * -1);  
                - reseller.resellerAccountBalance = encrypted(decrypted(reseller.resellerAccountBalance) + decrypted(resellerTransaction.transactionAmount)); 
                  _update reseller account balance_  
6. Case transaction_type = 721 and (order_id != NULL xor warehouse_order_id != NULL):  
   _cancel order_  
   _cancel warehouse order (positive transactions, refund)_
    - If order_id != NULL:
        - Get a list of warehouse order ids as warehouse_orders  
          WHERE warehouseOrder.orderID = order_id   
          AND warehouseOrder.warehouseOrderStatusID != 0;  
    - Else:  
        - warehouse_orders[0] = warehouse_order_id;  
    - For each warehouse_order_id in warehouse_orders:  
        - For each orderLineID in orderLine:  
          WHERE orderLine.warehouseOrderID = warehouse_order_id  
          AND orderLine.isDeleted = 0:
            - Calculate the weighted average price:
                - weighted_average_price = ABS(SUM(decrypted(resellerTransaction.transactionAmount)) / SUM(resellerTransaction.quantity)  
                  WHERE resellerTransaction.orderLineID = order_line_id  
                  AND resellerTransaction.transactionTypeID IN (711, 721));  
              - Create a record in resellerTransaction table where:  
                _transaction for cost of goods payment_  
                  - resellerTransaction.resellerID = website.resellerID   
                    WHERE website.websiteID = buyerAccount.websiteID  
                    AND buyerAccount.buyerAccountID = orders.buyerAccountID  
                    AND orders.orderID = order_id;  
                  - resellerTransaction.warehouseID = warehouseOrder.warehouseID;  
                  - resellerTransaction.orderID = order_id;  
                  - resellerTransaction.warehouseOrderID = warehouseOrder.warehouseOrderID;  
                  - resellerTransaction.orderLineID = orderLine.orderLineID;  
                  - resellerTransaction.transactionTypeID = 721;  
                  - resellerTransaction.quantity = encrypted(decrypted(orderLine.quantity) * -1);  
                  - resellerTransaction.warehouseProductPrice = encrypted(weighted_average_price);  
                  - resellerTransaction.transactionAmount = encrypted(decrypted(resellerTransaction.quantity) * decrypted(resellerTransaction.warehouseProductPrice) * -1);  
              - reseller.resellerAccountBalance = encrypted(decrypted(reseller.resellerAccountBalance) +decrypted(resellerTransaction.transactionAmount));  
              - Create a record in resellerTransaction table where:  
                _transaction for marketplace fee_
                  - resellerTransaction.resellerID = website.resellerID  
                      WHERE website.websiteID = buyerAccount.websiteID  
                      AND buyerAccount.buyerAccountID = orders.buyerAccountID  
                      AND orders.orderID = order_id;  
                  - resellerTransaction.warehouseID = warehouseOrder.warehouseID;  
                  - resellerTransaction.orderID = order_id;  
                  - resellerTransaction.warehouseOrderID = warehouseOrder.warehouseOrderID;  
                  - resellerTransaction.orderLineID = orderLine.orderLineID;  
                  - resellerTransaction.transactionTypeID = 722;  
                  - resellerTransaction.quantity = encrypted(decrypted(orderLine.quantity) * -1);  
                  - resellerTransaction.warehouseProductPrice = encrypted(weighted_average_price);  
                  - resellerTransaction.transactionAmount = encrypted(decrypted(resellerTransaction.quantity) * decrypted(resellerTransaction.warehouseProductPrice) * decrypted(reseller.resellerMarketplaceFee) * -1);  
                - reseller.resellerAccountBalance = encrypted(decrypted(reseller.resellerAccountBalance)  +resellerTransaction.transactionAmount);
