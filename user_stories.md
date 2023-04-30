# #01 User can add a new medical license

**AS** an operator  
**I WANT** to add a new buyer medical license  
**SO THAT** I can process process orders for this buyer

# Acceptance criteria
1. **SCENARIO: Succesfull license adding**
    - **GIVEN** buyer exists in the system
      - AND there is no license with same credentials
      - AND expiry date > todays date
    - **WHEN** user selects buyer
      - AND user enters medical license credentials
      - AND user enters license expiry date
    - **THEN** system saves medical license
2. **SCENARIO: Duplicated license**
    - **GIVEN** license with credentials 'xxxxxxxx' exists in the system
    - **WHEN** user add a new license with credentials 'xxxxxxxx'
    - **THEN** system return an error with message 'This license already exists in the system'
3. **SCENARIO: Invalid credentials**
    - **WHEN** user adds a new license with credentials less or more than 8 symbols
    - **THEN** system return an error with message 'Wrong credentials format'
4. **SCENARIO: Invalid expiry date**
    - **WHEN** user adds a new license with expiry date < todays date
    - **THEN** system return an error with message 'Expired license can't be added'
5. **SCENARIO: Missing credentials**
    - **WHEN** user adds a new license without credentials
    - **THEN** system return an error with message 'Missing license credentials'
6. **SCENARIO: Missing expiry date**
    - **WHEN** user adds a new license without expiry date
    - **THEN** system return an error with message 'Missing license expiry date'

---

# #02 Feature: User can submit a new order

**AS** an operator  
**I WANT** to submit a new order  
**SO THAT** I can manage it further  

# Relationships
1. Depends on #01 (necessity)

# Acceptance criteria

1. **SCENARIO: Successful order submission**
    - **GIVEN** requested products are exist
      - AND user role is operator
      - AND buyer exists in the system
      - AND VALID. asdasdadasdad.       medical license exists in the system
      - AND shipping address exists in the system
      - AND payment details exists in the system
      - AND buyer has bonuses
    - **WHEN** user selects product
      - AND user selects product quantity
      - AND user selects medical license
      - AND user selects shipping address
      - AND user selects payment details
      - AND user applies bonuses
      - AND user submit selections
    - **THEN** system should save order
2. **SCENARIO: Missing product**
    - **GIVEN** user conducts order submission
      - AND requested product is missing
    - **WHEN** user selects product
      - AND user can’t find product
    - **THEN** user should be able to cancel order processing
3. **SCENARIO: Missing medical license**
    - **GIVEN** user conducts order submission
      - AND medical license is missing
    - **WHEN** user selects medical license
      - AND can't find medical license
    - **THEN** user should be able to add a new medical license
4. **SCENARIO: Missing shipping address**
    - **GIVEN** user conducts order submission
      - AND shipping address is missing
    - **WHEN** user selects shipping address
      - AND can't find shipping address
    - **THEN** user should be able to add a new shipping address
5. **SCENARIO: Missing payment details**
    - **GIVEN**  user conducts order submission
      - AND payment details are missing
    - **WHEN** user selects payment details
      - AND can't find payment details
    - **THEN** user should be able to add a new payment details
6. **SCENARIO: Buyer doesn't have bonuses**
     - **GIVEN**  user conducts order submission
        - AND user doesn't have bonuses
    - **THEN** system should skip bonuses step
7. **SCENARIO: Buyer has bonuses, but don't apply them**
     - **GIVEN**  user conducts order submission
        - AND user has bonuses
    - **WHEN** user don't apply bonuses
    - **THEN** system should skip bonuses step 
---

# #03 System store sensetive buyer information in encypted format
