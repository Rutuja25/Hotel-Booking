Hotel Booking System — Interview Problem Statement

“Design and implement a Hotel Booking System.”

The system should allow customers to search for hotels based on location and dates, view available rooms, and book a room.

A hotel can have multiple rooms, and rooms can have different room types such as Single, Double, Deluxe, and Suite.

Customers should be able to:

Search for hotels in a particular location.
Search for available rooms for a given check-in and check-out date.
View room details and pricing.
Book an available room.
Cancel an existing booking.
View their bookings.

The system should prevent two customers from booking the same room for overlapping dates.

Different hotels may have different pricing strategies, and the system should be designed so that new pricing strategies can be added easily in the future.

Design the classes, interfaces, and relationships required to implement this system.

Then the interviewer may add requirements one by one

After you've started your design, expect follow-ups such as:

Interviewer:

“A hotel can have multiple rooms of the same type. How would you model that?”

Interviewer:

“What happens if two customers try to book the same room at exactly the same time?”

Interviewer:

“We want weekend pricing to be different from weekday pricing. How would you design that?”

Interviewer:

“Now introduce seasonal pricing.”

Interviewer:

“Some rooms can be cancelled for a full refund, while others have a cancellation fee. How would you model that?”

Interviewer:

“What if a customer wants to book three rooms in one reservation?”

Interviewer:

“Suppose we want to support different payment methods—credit card, UPI, PayPal, etc.”

Interviewer:

“How would you modify the design if we later add apartments or villas?”
