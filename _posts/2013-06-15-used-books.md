---
layout: post
title:  "Automating Used Book Inventories"
date:   2013-06-15 09:36:54
categories: shelved projects
tags: meh
image: http://i.imgur.com/kSmtbDI.jpg
---

I love books. Really love books. New, used, tattered, torn, I don't care. Books are great.

My wife and I buy the biggest quantity of books from used book stores. The greatest thing for us about used book stores is that they **always** have a decent section of children's books (which are always timeless). We end up buying some new used books at [ReReads](http://rereadsonline.com/) for our daughter and kids in the neighborhood whenever we visit, since most are around one or two dollars. We're moving to San Antonio this week, so we look forward to [Half Price Books](http://www.hpb.com) as it appears to have a streamlined process, an active site and decent prices.

Technical books, the books I really need, are typically purchased on Amazon. Sometimes I'll get them from a brick and mortar store if the price is right or I want it *right now*. Used book stores almost never satisfy my craving for technical books, as I have no desire to buy books about Office 97.

There is one glaring issue we have seen across many of the smaller used book stores though.

# Piles of Books

![Piles of Books](http://i.imgur.com/kSmtbDI.jpg)

Many used book stores have piles of books but don't appear to have a good system for entering them into a system, putting inventories online and sometimes don't have a way to display all of their books. There are stores/sites that do (Powell's, Half Price Books, and partially Amazon and their resellers), but all the small shops look like they're being crushed under piles of books.

*We want those books moving, fluidly, into the hands of other avid readers.*

Software exists to help with [inventory and resale](http://www.abebooks.com/homebase/software-inventory-management-system-catalog/), but it doesn't go all the way and certainly isn't open source.

# Used Book Ingest

If we started an idealized nonprofit used bookstore, this is how we would ingest and sell books:

* **Donors scan their own books** into a simple terminal

* **Donors pick the price** for others to buy it from the store

* **Retrieve book information from the web** and allow the donor to fix the entry

* **Give donors an itemized receipt of their donation**. As a bonus, tell them where to put the books

* After ingest, **place books online immediately**

There are some clearcut software components here and a slight picture of a hardware stack. So as not to subject myself completely to Not Invented Here syndrome, I did a little bit of digging around. There was a clear market for consignment software, but a lot of it was fairly niche or outdated. The one exception, as mentioned above, was [Abe books](http://www.abebooks.com/homebase/software-inventory-management-system-catalog/). They had the best software that wasn't completely coupled to their own systems or living in the nineties.

However, as an open source zealot none of these match what I want for the future, which is community maintained software that anyone can contribute to. We'd like to enable used book stores, co-ops, new book stores, really anyone wanting to track inventory of books with ISBNs.

My wife and I would love to start a nonprofit used book store and coffee shop, but we're not ready for that yet. We'd love to start building the software and work towards this side goal.

If you want to join us on this adventure, hit me up on [Twitter](http://twitter.com/rgbkrk/).
