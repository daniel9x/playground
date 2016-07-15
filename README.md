<center>

# Event Management System (EMS) Service

<font face="Arial" size="4">**Design Overview**</font>

Ben Torkian / Daniel Grant  
Last Updated: July 14, 2016

**- C O N F I D E N T I A L -**

## Build Steps

1. In order to run EMS locally, you will need a local instance of a Postgres 9.5 Database running on your machine.
2. The backing EMS Database consists of 3 Tables.
|   |
|---|
|   |
|   |
|   |



## Overview

The Event Management System is entirely event driven, as the name suggests. Therefore, it is easiest to describe its fuctional design my describing an (ordered) sequence of actions EMS performs.

1.  **EMS** receives (UBD) messages via a RabbitMQ queue that the Summarist Service publishes to.

3.  **EMS** queries its **Rule** table to determine if there is a "matching rule" for the particular message it just received.

5.  If a match is found, it queries **Summarist** for the related message to compare with. If no match is found, the message is discarded.

7.  **EMS** takes the two correlated messages and applies the comparison logic defined in the rule.

9.  **EMS** persists the result of the comparison into the **Rule Result** table. _The following section goes into more detail on this._

11.  **EMS** uses the **Milestone** table to determine how these results are ordered and presented when queries by the Visibility application.

![](images/Capture.jpg)

## Chaining The Rule Results Together

*   Each **Rule Result** record has an (optional) reference to a previous rule result to allow for chaining.

*   When **EMS** queries **Summarist** for a **Related UBD** (_Step 3 above_), it will (_in Step 5)_ check to see if that **Related UBD** has been previously processed as a (main) **EventUBD.**

*   If the condition above returns **true**, it chains the current result to the previous result. Click [here](http://prezi.com/gagzgrqiu_0v/?utm_campaign=share&utm_medium=copy&rc=ex0share) to see an animation illustrating this.

## Scenarios

1.  OrderCreate / Partner Pair Match

    1.  **Summarist** publishes an **Order Create** from Evonik to BASF to the **EMS** Queue.
    2.  **EMS** consumes the messages on the queue and checks to see if there is a rule for the message type (**Order Create**) and the sender/receiver (BASF/Evonik).
    3.  **EMS** finds a matching rule and checks for the related message type(s) on the rule.
    4.  The matching rule does not have a related message type. No comparison logic is appled, but the (mostly empty) result is persisted.
2.  OrderCreate / Non-Partner Pair Match

    1.  **Summarist** publishes an **Order Create** from Evonik to BASF to the **EMS** Queue.
    2.  **EMS** consumes the messages on the queue and checks to see if there is a rule for the message type (**Order Create**) and the sender/receiver (BASF/Evonik).
    3.  **EMS** does not find a match. **EMS** checks to see if there is a matching rule for the message type (**Order Create**) and the sender (Evonik).
    4.  **EMS** finds a matching rule and checks for the related message type(s) on the rule.
    5.  The matching rule does not have a related message type. No comparison logic is appled, but the (mostly empty) result is persisted.
3.  OrderResponse / Order Create Match

    1.  **Summarist** publishes an **Order Response** from BASF to Evonik to the **EMS** Queue.
    2.  **EMS** consumes the messages on the queue and checks to see if there is a rule for the message type (**Order Response**) and the sender/receiver (Evonik/BASF).
    3.  **EMS** finds a matching rule and checks for the related message type(s) on the rule.
    4.  **EMS** compiles the Identifier Keys for the specific rule and the message types. In this case, the Identifier Keys are the PO Number and PO Line Number. The Related Message types are **Order Create** and Order Change.
    5.  **EMS** swaps the Sender/Receiver pair based on the rule configuration.
    6.  **EMS** Queries **Summarist** with the swapped Sender/Receiver pair and the Identifier Keys for the most recent **Order Create** or **Order Change**.
    7.  **Order Create** is returned. The Rule comparison logic is applied.
    8.  **EMS** checks contents of warnings generated as a result of the comparisons and determines which notification to send out based on whether warnings were generated.
    9.  **EMS** queries its own Rule Result table for a matching result with the **Order Create**'s UUID. It finds a match, and ties the **Order Response** to the Previous Result.
