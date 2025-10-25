# Lab 6 Fix Order Pricing and Implement Password Reset Emails

### Step 3: Fix Order Pricing

1. Create a contract in Products.Contracts for getting product pricing
2. Implement a handler in the Products module to return product prices
3. Update the Order creation logic to fetch prices from Products module
4. Remove price parameters from Order-related endpoints and commands
5. Add an endpoint to confirm an Order; it should change the Order status to Processing.
5. Email the user when their Order has been placed/confirmed (send command to email module)
6. Raise an OrderCreatedEvent, which should be defined in the Customers.Contracts project, and should include details on the Order and all of its Items (Product Ids, Quantities, and Amounts).

### Step 4: Implement Password Generation

1. Add a password generation utility
2. Update Customer creation to generate a random password
3. Store the password securely using Identity's password hasher
4. Send the generated password via email (command to email module)

### Step 5: Implement Password Reset

1. Create a password reset endpoint
2. Generate a new random password
3. Update the user's password
4. Email the new password to the user (command to email module)


## Success Criteria for Lab 6

After completing this lab, you should have:

- ✅ Orders that fetch prices from the Products module
- ✅ Automatic password generation for new customers
- ✅ Password reset functionality with email notification

## Notes

- For simplicity new passwords can just be a portion of a GUID
- A real app should have password reset tokens with expiration for better security
