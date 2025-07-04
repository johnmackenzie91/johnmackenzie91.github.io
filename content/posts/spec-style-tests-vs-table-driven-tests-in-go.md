+++
title = 'Spec Style Tests vs Table Driven Tests in Go'
date = 2025-07-04T20:10:59+01:00
draft = true
+++

### Spec-Style Tests vs Table-Driven Tests in Go

When unit testing in Go, the first style of test that springs to mind is usually table-driven tests. They’re idiomatic and concise — `for *this* input, expect *this* output`. However, for certain kinds of functionality, table tests can become repetitive and hard to read.

I’ve found that **spec-style tests** can be a better fit when testing things like:

- Complex API handlers

- Integration tests (e.g. database clients, API clients)


---

### The Problem with Table Tests for Integration-Like Scenarios

Imagine an integration test suite for a data access layer written using the traditional table test pattern:

```go
func Test_ListIntervals(t *testing.T) {
	testCases := []struct{
		name string
		args ListIntervalOption...
		expectedOutput []Interval
		expectedErr error
	}{
		// ...
	}
	for _, tc := range testCases {
		t.Run(tc.name, func(t *testing.T){
			// ...
		})
	}
}
```

Here’s the issue: let’s say we populate the database with test data. For each test case, we now need to explicitly define the expected output — often duplicating the same dataset or structures across cases. This leads to **more boilerplate** and **higher cognitive load** when reading or maintaining the test suite.

---

### A Spec-Style Alternative

Now consider the same logic tested using a spec-style approach:

```go
func Test_ListIntervals(t *testing.T) {  
    sut := setup(t)

    t.Run("Given the database is empty, no intervals are returned", func(t *testing.T) {  
        got, err := sut.List(context.Background())  
        require.NoError(t, err)  
        require.Empty(t, got)  
    })  

    t.Run("Given there are 6 intervals in the database", func(t *testing.T) {  
        _, err := sut.DB.Exec(`INSERT INTO intervals (uuid, start, end) VALUES  
        ('cf32...', '2025-07-01T10:00:00Z', '2025-07-01T11:00:00Z'),
        ('3bb2...', '2025-07-02T10:00:00Z', '2025-07-02T11:00:00Z'),
        -- and so on --
        `)  
        require.NoError(t, err)

        t.Run("Then a default limit of 5 is applied", func(t *testing.T) {  
            got, err := sut.List(context.Background())  
            require.NoError(t, err)  
            require.Len(t, got, 5)  
        })  

        t.Run("Then the default sort order is descending", func(t *testing.T) {  
            got, err := sut.List(context.Background())  
            require.NoError(t, err)  
            for i := 0; i < len(got)-1; i++ {  
                require.True(t, got[i].Start.After(got[i+1].Start) || got[i].Start.Equal(got[i+1].Start))  
            }  
        })  

        t.Run("When a limit of 6 is set, 6 intervals are returned", func(t *testing.T) {  
            got, err := sut.List(context.Background(), WithLimit(6))  
            require.NoError(t, err)  
            require.Len(t, got, 6)  
        })  

        t.Run("When order is ascending, intervals are ordered correctly", func(t *testing.T) {  
            got, err := sut.List(context.Background(), WithOrder("ASC"))  
            require.NoError(t, err)  
            for i := 0; i < len(got)-1; i++ {  
                require.True(t, got[i].Start.Before(got[i+1].Start) || got[i].Start.Equal(got[i+1].Start))  
            }  
        })  

        t.Run("When afterID is set, only later intervals are returned", func(t *testing.T) {  
            got, err := sut.List(  
                context.Background(),  
                WithAfterID(uuid.MustParse("3bb2f81e-...")),  
            )  
            require.NoError(t, err)  
            assert.Len(t, got, 1)  
            assert.Equal(t, uuid.MustParse("cf32d8bb-..."), got[0].ID)  
        })  
    })  
}
```

This approach is **more concise and readable**:

- We set up fixtures once, instead of per-case.
- Assertions test _properties_ (e.g. length, ordering) rather than explicit arrays.
- The structure reflects the actual behaviour of the system rather than abstract inputs/outputs.

This also means tests are **less tightly coupled to implementation details**, reducing the need to update them when internal logic changes slightly.

---

### Why Spec-Style Tests Work Better Here

Spec-style tests emphasise **readability** and **behavioural clarity**, which is exactly what you want when dealing with systems that have:

- complex setup
- branching logic
- implicit state (like databases or sessions)

They also double as **living documentation**. And let’s be honest — descriptive test names in plain English often communicate more than even the most elegant table test.

---

### So When Should You Use Each?

|Style|Best For|
|---|---|
|**Table Tests**|Pure functions, simple HTTP handlers, validation, data transformations|
|**Spec Tests**|Complex handlers, behaviour-driven testing, integration layers|

---

### Final Thoughts

Go’s testing culture is table-test heavy — for good reason. But don’t be afraid to break out of that pattern when it starts working _against_ you. Spec-style tests can make your tests more maintainable, more expressive, and ultimately, more helpful.