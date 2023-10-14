---
title: "Unit testing using Golang - a beginner's guide"
description: "A beginners guide to writing unit tests in Golang · Let's first start with what unit testing is. Unit Testing is a way of testing where the behavior of a..."
dateString: October 2023
draft: false
tags: ["Golang", "Testing", "Unit-Testing", "Open-source"]
weight: 102
cover:
    image: "blog/testing.avif"
---

# Unit testing using Golang

Let's first start with what unit testing is. Unit Testing is a way of testing where the behavior of a unit of software is tested to determine if it works properly and exhibits the expected behavior. But why do we need unit testing? There are multiple reasons for that. See, how can we ensure that even if we change code, the code still works and there are no new bugs introduced, it's better to detect them early than to wait and run the entire application in order to check a small code. In this way, unit tests help ensure quality code, help in catching bugs early, and provide a safety net if the code changes in the future.

## What is unit testing?

As per IEEE (Institute for Electrical and Electronics Engineers), unit testing is the “testing of individual hardware or software units or groups of related units”. Unit testing is a software testing technique where individual components or units of a software application are tested in isolation.

By unit, we mean the smallest testable part of a code. mostly, it is a single function in Golang. The goal of unit testing is to ensure that each of these units functions correctly (produces expected results).

The term "unit" in unit testing refers to the smallest testable part of a software program, typically a single function, method, or class. The primary goal of unit testing is to ensure that each of these individual units of code functions correctly and produces the expected results.

## Why Unit tests in Go?

• Helps in verifying small changes (refactoring, debugging) quickly • Measures the quality of the code

• Helps in understanding the complex logics

• Helps in understanding the cause of failure quickly

• Helps the reviewer to understand the fixes/changes

• Great way to learn about a language as well as the project.

## Let's create a basic unit test:

**Step 1.** Start with initializing a go package

`go mod init github.com/1shubham7/basic-unit-test`

**Step 2.** Download a unit testing package called go-money

`go get github.com/Rhymond/go-money`

**Step 3.** Create this file structure:

![a23644aa-ded1-4755-8ef1-79c75cd5b136](https://github.com/1Shubham7/project-currency/assets/116020663/6bac4247-0a73-4d3d-a61d-1f087c952d49)

Create a internal folder, inside that create a order folder, and inside the order folder, create a order.go file.

**Step 4.** Start coding in the order.go file

```
package order

import (
	"github.com/Rhymond/go-money"
	"fmt"
)

type Order struct {
	ID                string
	CurrencyAlphaCode string
	Items             []Item
}

type Item struct {
	ID        string
	Quantity  uint
	UnitPrice *money.Money
}

func (o Order) ComputeTotal() (*money.Money, error) {
	amount := money.New(0, o.CurrencyAlphaCode)
	for _, item := range o.Items {
		var err error
		amount, err = amount.Add(item.UnitPrice)
		if err!= nil {
			return nil, fmt.Errorf("not adding item elements, error: %w", err)
		}
	}

	return amount, nil
}
```


**Step 5.** Enter the following command to download testify

`/go get github.com/stretchr/testify`

**Step 6.** Create a order_test.go file and enter the following code inside it:

```
package order

import (
	"testing"

	"github.com/Rhymond/go-money"

	"github.com/stretchr/testify/assert"
)

func TestOrder(t *testing.T) {
	o := Order{
		ID: "100",
		CurrencyAlphaCode: "INR",
		Items: []Item{
			{
			ID: "500",
			Quantity: 2,
			UnitPrice: money.New(100, "INR"),
		},
	},
}
	total, err := o.ComputeTotal()
	assert.NoError(t, err)
	assert.Equal(t, 200, total.Amount())
	assert.Equal(t, "INR", total.Currency())
}
```

**Step 7.** Now this command is used to run all the unit test in that perticular directory, enter the following command:

`go test ./...`

it will give you the result:

```
--- FAIL: TestOrder (0.00s) order_test.go:25: Error Trace: D:/Files/Golang/Basic Unit Test/internal/order/order_test.go:25
Error: Not equal: expected: int(200) actual : int64(100) Test: TestOrder order_test.go:26: Error Trace: D:/Files/Golang/Basic Unit Test/internal/order/order_test.go:26
Error: Not equal: expected: string("INR") actual : *money.Currency(&money.Currency{Code:"INR", NumericCode:"356", Fraction:2, Grapheme:"₹", Template:"$1", Decimal:".", Thousand:","}) Test: TestOrder FAIL FAIL github.com/1shubham7/basic-unit-test/internal/order 1.001s FAIL
```

**Step 8.** to fix this up, we have to make some changes in the code:

Make these changes to the code:

***order.go:***


package order

```
import (
	"github.com/Rhymond/go-money"
	"fmt"
)

type Order struct {
	ID                string
	CurrencyAlphaCode string
	Items             []Item
}

type Item struct {
	ID        string
	Quantity  uint
	UnitPrice *money.Money
}

func (o Order) ComputeTotal() (*money.Money, error) {
	amount := money.New(0, o.CurrencyAlphaCode)
	for _, item := range o.Items {
		var err error
		amount, err = amount.Add(item.UnitPrice.Multiply(int64(item.Quantity)))
		if err!= nil {
			return nil, fmt.Errorf("not adding item elements, error: %w", err)
		}
	}

	return amount, nil
}
```

***order_test.go:***

```
package order

import (
	"testing"

	"github.com/Rhymond/go-money"

	"github.com/stretchr/testify/assert"
)

func TestOrder(t *testing.T) {
	o := Order{
		ID: "100",
		CurrencyAlphaCode: "INR",
		Items: []Item{
			{
			ID: "500",
			Quantity: 2,
			UnitPrice: money.New(100, "INR"),
		},
	},
}
	total, err := o.ComputeTotal()
	assert.NoError(t, err)
	assert.Equal(t, int64(200), total.Amount())
	assert.Equal(t, "INR", total.Currency().Code)
}
```

Now if you enter the same command:

`go test ./...`

It works perfectly fine:
![image](https://github.com/1Shubham7/project-currency/assets/116020663/b2c0cf06-6d50-4de9-b3fd-576643e739cc)

And that's how we create a unit test. See you in some next tutorial.
Thanks for reading.
