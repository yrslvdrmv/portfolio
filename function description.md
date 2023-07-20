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
7. Case transaction_type = 711 and order_line_id != NULL and quantity != NULL:
# cancel warehouse order https://statystech.atlassian.net/browse/OMS-957 
# move order line https://statystech.atlassian.net/browse/LWA-1265
Create a record in resellerTransaction table where: 
# transaction for cost of goods payment
resellerTransaction.resellerID = website.resellerID 
WHERE website.websiteID = buyerAccount.websiteID
AND buyerAccount.buyerAccountID = orders.buyerAccountID
AND orders.orderID = warehouseOrder.orderID
AND warehouseOrder.warehouseOrderID = orderLine.warehouseOrderID
AND orderLine.orderLineID = order_line_id;
resellerTransaction.warehouseID = warehouseOrder.warehouseID;
resellerTransaction.orderID = order_id;
resellerTransaction.warehouseOrderID = warehouseOrder.warehouseOrderID;
resellerTransaction.orderLineID = order_line_id;
resellerTransaction.transactionTypeID = 711;
resellerTransaction.quantity = encrypted(quantity);
If reseller.currencyCD == warehouse.currencyCD:
resellerTransaction.currencyExchangeRate = 1;
Else:
resellerTransaction.currencyExchangeRate = currencyExchangeRate.exchangeRate * (1+currencyExchangeRate.marketplaceMarkup)
WHERE currencyExchangeRate.fromCurrencyCD = warehouse.currencyCD
AND currencyExchangeRate.toCurrencyCD = reseller.currencyCD
ORDER BY currencyExchangeRate.fxDate DESC
LIMIT 1;
If warehouseInventory.warehouseProductPrice == NULL
WHERE warehouseInventory.warehouseID = warehouseOrder.warehouseID 
AND warehouseInventory.productID = orderLine.productID:
resellerTransaction.warehouseProductPrice = encrypted(-1);
Ese:
resellerTransaction.warehouseProductPrice = encrypted(warehouseInventory.warehouseProductPrice * resellerTransaction.currencyExchangeRate);
resellerTransaction.transactionAmount = encrypted(decrypted(resellerTransaction.quantity) * decrypted(resellerTransaction.warehouseProductPrice) * -1);
reseller.resellerAccountBalance = encrypted(decrypted(reseller.resellerAccountBalance) + decrypted(resellerTransaction.transactionAmount)); 
# update reseller account balance
Create a record in resellerTransaction table where: 
# transaction for marketplace fee
resellerTransaction.resellerID = website.resellerID 
WHERE website.websiteID = buyerAccount.websiteID
AND buyerAccount.buyerAccountID = orders.buyerAccountID
AND orders.orderID = warehouseOrder.orderID
AND warehouseOrder.warehouseOrderID = orderLine.warehouseOrderID
AND orderLine.orderLineID = order_line_id;
resellerTransaction.warehouseID = warehouseOrder.warehouseID;
resellerTransaction.orderID = order_id;
resellerTransaction.warehouseOrderID = warehouseOrder.warehouseOrderID;
resellerTransaction.orderLineID = order_line_id;
resellerTransaction.transactionTypeID = 712;
resellerTransaction.quantity = encrypted(quantity);
If reseller.currencyCD == warehouse.currencyCD:
resellerTransaction.currencyExchangeRate = 1;
Else:
resellerTransaction.currencyExchangeRate = currencyExchangeRate.exchangeRate * (1+currencyExchangeRate.marketplaceMarkup)
WHERE currencyExchangeRate.fromCurrencyCD = warehouse.currencyCD
AND currencyExchangeRate.toCurrencyCD = reseller.currencyCD
ORDER BY currencyExchangeRate.fxDate DESC
LIMIT 1;
If warehouseInventory.warehouseProductPrice == NULL
WHERE warehouseInventory.warehouseID = warehouseOrder.warehouseID 
AND warehouseInventory.productID = orderLine.productID:
resellerTransaction.warehouseProductPrice = encrypted(-1);
Ese:
resellerTransaction.warehouseProductPrice = encrypted(warehouseInventory.warehouseProductPrice * resellerTransaction.currencyExchangeRate);
resellerTransaction.transactionAmount = encrypted(decrypted(resellerTransaction.quantity) * decrypted(resellerTransaction.warehouseProductPrice) * decrypted(reseller.resellerMarketplaceFee) * -1);
reseller.resellerAccountBalance = encrypted(decrypted(reseller.resellerAccountBalance) + decrypted(resellerTransaction.transactionAmount)); 
# update reseller account balance
8. Case transaction_type = 721 and (order_line_id != NULL xor parcel_line_id != NULL) and quantity != NULL:
# cancel order line https://statystech.atlassian.net/browse/LWA-1266 
# move order line https://statystech.atlassian.net/browse/LWA-1265
# reship/refund lost parcel https://statystech.atlassian.net/browse/OMS-948
Calculate the weighted average price:
weighted_average_price = ABS(SUM(decrypted(resellerTransaction.transactionAmount)) / SUM(resellerTransaction.quantity)
WHERE resellerTransaction.orderLineID = order_line_id
AND resellerTransaction.transactionTypeID IN (711, 721));
If parcel_line_id != NULL:
order_line_id = parcelLine.orderLineID
WHERE parcelLineID = parcel_line_id;
Create a record in resellerTransaction table where: 
# transaction for cost of goods payment
resellerTransaction.resellerID = website.resellerID 
WHERE website.websiteID = buyerAccount.websiteID
AND buyerAccount.buyerAccountID = orders.buyerAccountID
AND orders.orderID = warehouseOrder.orderID
AND warehouseOrder.warehouseOrderID = orderLine.warehouseOrderID
AND orderLine.orderLineID = order_line_id
resellerTransaction.warehouseID = warehouseOrder.warehouseID;
resellerTransaction.orderID = order_id;
resellerTransaction.warehouseOrderID = warehouseOrder.warehouseOrderID;
resellerTransaction.orderLineID = order_line_id if order_line_id != NULL else: parcelLine.orderLineID WHERE parcelLine.parcelLineID = parcel_line_id;
resellerTransaction.parcelLineID = parcel_line_id if parcel_line_id != NULL;
resellerTransaction.parcelID = parcelLine.parcelID WHERE parcelLine.parcelLineID = parcel_line_idif parcel_line_id != NULL;
resellerTransaction.transactionTypeID = 721;
resellerTransaction.quantity = encrypted(quantity * -1);
resellerTransaction.warehouseProductPrice = encrypted(weighted_average_price);
resellerTransaction.transactionAmount = encrypted(decrypted(resellerTransaction.quantity) * decrypted(resellerTransaction.warehouseProductPrice) * -1);
reseller.resellerAccountBalance = encrypted(decrypted(reseller.resellerAccountBalance) + decrypted(resellerTransaction.transactionAmount)); 
Create a record in resellerTransaction table where: 
# transaction for marketplace fee
resellerTransaction.resellerID = website.resellerID 
WHERE website.websiteID = buyerAccount.websiteID
AND buyerAccount.buyerAccountID = orders.buyerAccountID
AND orders.orderID = order_id;
resellerTransaction.warehouseID = warehouseOrder.warehouseID;
resellerTransaction.orderID = order_id;
resellerTransaction.warehouseOrderID = warehouseOrder.warehouseOrderID;
resellerTransaction.orderLineID = order_line_id if order_line_id != NULL else: parcelLine.orderLineID WHERE parcelLine.parcelLineID = parcel_line_id;
resellerTransaction.parcelLineID = parcel_line_id if parcel_line_id != NULL;
resellerTransaction.parcelID = parcelLine.parcelID WHERE parcelLine.parcelLineID = parcel_line_id if parcel_line_id != NULL;
resellerTransaction.transactionTypeID = 722;
resellerTransaction.quantity = encrypted(quantity * -1);
resellerTransaction.warehouseProductPrice = encrypted(weighted_average_price);
resellerTransaction.transactionAmount = encrypted(decrypted(resellerTransaction.quantity) * decrypted(resellerTransaction.warehouseProductPrice) * decrypted(reseller.resellerMarketplaceFee) * -1);
reseller.resellerAccountBalance = encrypted(decrypted(reseller.resellerAccountBalance) + decrypted(resellerTransaction.transactionAmount)); 
9. Case transaction_type = 713 and parcel_id != NULL:
# external fulfillment (charge shipping cost) https://statystech.atlassian.net/browse/LWA-1319
Create a record in resellerTransaction table with:
resellerTransaction.resellerID = website.resellerID
WHERE resellerTransaction.resellerID = website.resellerID 
AND website.websiteID = buyerAccount.websiteID
AND buyerAccount.buyerAccountID = orders.buyerAccountID
AND orders.orderID = warehouseOrder.orderID
AND warehouseOrder.warehouseOrderID = parcel.warehouseOrderID
AND parcel.parcelID = parcel_id
resellerTransaction.orderID = orders.orderID;
resellerTransaction.warehouseID = warehouseOrder.warehouseID;
resellerTransaction.warehouseOrderID = warehouseOrder.warehouseOrderID;
resellerTransaction.parcelID = parcel_id;
resellerTransaction.transactionTypeID = 713; 
resellerTransaction.warehouseProductPrice = 0;
resellerTransaction.quantity = 0;
If at least one product in parcel_id have productParent.isCold == 1
WHERE productParent.productParentID = product.productParentID 
AND product.productID = orderLine.productID 
AND orderLine.orderLineID = parcelLine.orderLineID
AND parcelLine.parcelID = parcel_id: 
# case when parcel is cold chain
If reseller.currencyCD == ‘USD’: 
# shipping rates are always in USD, case when reseller currency == USD
resellerTransaction.currencyExchangeRate = 1;
resellerTransaction.transactionAmount = encrypted(shippingRate.coldChainRate * -1);
Else: 
# shipping rates are always in USD, case when reseller currency != USD
resellerTransaction.currencyExchangeRate = currencyExchangeRate.exchangeRate * (1+currencyExchangeRate.marketplaceMarkup)
WHERE currencyExchangeRate.fromCurrencyCD = “USD”
AND currencyExchangeRate.toCurrencyCD = reseller.currencyCD
ORDER BY currencyExchangeRate.fxDate DESC
LIMIT 1;
resellerTransaction.transactionAmount = encrypted(shippingRate.coldChainRate * resellerTransaction.currencyExchangeRate * -1);
Else:
# case when parcel is ambient
If reseller.resellerCurrencyCD == ‘USD’: 
# shipping rates are always in USD, case when reseller currency == USD
resellerTransaction.currencyExchangeRate = 1;
resellerTransaction.transactionAmount = encrypted(shippingRate.ambientRate * -1)
WHERE shippingRate.warehouseID = warehouseOrder.warehouseID;
Else: 
# shipping rates are always in USD, case when reseller currency != USD
resellerTransaction.currencyExchangeRate = currencyExchangeRate.exchangeRate * (1+currencyExchangeRate.marketplaceMarkup)
WHERE currencyExchangeRate.fromCurrencyCD = “USD”
AND currencyExchangeRate.toCurrencyCD = reseller.currencyCD
ORDER BY currencyExchangeRate.fxDate DESC
LIMIT 1;
resellerTransaction.transactionAmount = encrypted(ambientRate.coldChainRate * resellerTransaction.currencyExchangeRate * -1);
reseller.resellerAccountBalance = encrypted(decrypted(reseller.resellerAccountBalance) + decrypted(resellerTransaction.transactionAmount)); 
10. Case transaction_type = 723 and parcel_id != NULL:
# Delete parcel https://statystech.atlassian.net/browse/LWA-1318 
# Reset order https://statystech.atlassian.net/browse/LWA-1320
Find a record in resellerTransaction table
WHERE resellerTransaction.transactionTypeCD == 713
AND resellerTransaction.parcelID = parcel_id;
Based on record 12.a create new one with:
resellerTransaction.transactionTypeID = 723;
resellerTransaction.transactionAmount = encrypted(decrypted(resellerTransaction.transactionAmount) * -1);
Set reseller.resellerAccountBalance = encrypted(decrypted(reseller.resellerAccountBalance) + resellerTransaction.transactionAmount(723));

