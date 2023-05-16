# Description
API submit a new order to the system

# Request

## Endpoint
```POST /v1/orders```
## Parameters
```products```:  
$~~~~~~~~~$ A list of products  
$~~~~~~~~~$ required: yes  
$~~~~~~~~~$ type: array of objects  
$~~~~~~~~~$ max length: 50  
```product_id```:  
$~~~~~~~~~$ id of product  
$~~~~~~~~~$ required: yes  
$~~~~~~~~~$ type: string  
```quantity```:  
$~~~~~~~~~$ requested product quantity  
$~~~~~~~~~$ required: yes  
$~~~~~~~~~$ type: string  
```buyer```:  
$~~~~~~~~~$ Buyer details  
$~~~~~~~~~$ required: yes  
$~~~~~~~~~$ type: object  
```buyer_id```:  
$~~~~~~~~~$ id of buyer  
$~~~~~~~~~$ required: yes  
$~~~~~~~~~$ type: string  
```website_id```:  
$~~~~~~~~~$ id of website  
$~~~~~~~~~$ required: yes  
$~~~~~~~~~$ type: string  
```shipping_address_id```:  
$~~~~~~~~~$ id of shipping address  
$~~~~~~~~~$ required: yes  
$~~~~~~~~~$ type: string  
```medical_license_id```:  
$~~~~~~~~~$ id of medical license  
$~~~~~~~~~$ required: yes  
$~~~~~~~~~$ type: string  
```payment_details_id```:  
$~~~~~~~~~$ id of payment information  
$~~~~~~~~~$ required: yes  
$~~~~~~~~~$ type: string  
## Input schema
```json
{
    "products": [
        {
            "product_id": "string",
            "quantity": "integer"
        }
    ],
    "buyer": {
        "buyer_id": "string",
        "website_id": "string",
        "shipping_address_id": "string",
        "medical_license_id": "string",
        "payment_details_id": "string"
    },
    "required": ["product_id", "quantity", "buyer_id", "website_id", "shipping_address_id", "medical_license_id", "payment_details_id"]
}
```

# Response
## Parameters
```order_id```:  
$~~~~~~~~~$ id of submited order  
$~~~~~~~~~$ type: string  
## Output schema

```json
{
    "order_id": "string"
}
```
# Errors

| HTTP Status Code | Message |
| ------------- | ------------- |
| 400  | Missing product id |
| 400  | Invalid product id |
| 400  | Missing quantity |
| 400  | Invalid quantity |
| 400  | Missing buyer id |
| 400  | Invalid buyer id |
| 400  | Missing website id |
| 400  | Invalid website id |
| 400  | Missing shipping address id |
| 400  | Invalid shipping address id |
| 400  | Missing medical license id |
| 400  | Invalid medical license id |
| 400  | Missing payment details id |
| 400  | Invalid payment details id |
| 403  | Access denied |
