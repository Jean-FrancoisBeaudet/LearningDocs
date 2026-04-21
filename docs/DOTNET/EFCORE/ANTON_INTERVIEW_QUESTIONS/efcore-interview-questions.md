# EF Core Interview Questions

Source: `net-interview-questions.pdf` by Anton Martyniuk (Microsoft MVP).

## Junior Level

1. What is a migration in EF Core?
2. How do you apply pending migrations to a database?
3. What is the difference between Eager Loading, Explicit Loading and Lazy Loading?
4. What are navigation properties and why are they important?
5. What is the difference between value types and reference types in EF Core mappings?
6. How do you configure relationships (one-to-many, many-to-many) in EF Core models?
7. How do you configure one-to-many relationships in EF Core?
8. How does EF Core handle cascade delete behavior?
9. What is the difference between tracked and untracked entities?
10. What does `AsNoTracking` do and when should you use it?

## Mid / Senior Level

11. What are shadow properties in EF Core?
12. How does EF Core handle concurrency conflicts?
13. How do you configure indexes using the Fluent API?
14. What is the difference between `FirstOrDefault` and `SingleOrDefault` in EF queries?
15. What is the purpose of `AsSplitQuery`?
16. How does EF Core translate LINQ expressions into SQL?
17. What are global query filters and when are they used?
18. What is the difference between `SaveChanges` and `SaveChangesAsync`?
19. How does EF Core handle enum properties in entities?
20. What are owned types in EF Core?
21. How do you seed data using EF Core migrations?
22. What is the difference between client-side and server-side evaluation?
23. How do you execute raw SQL queries in EF Core?
24. How do you design an aggregate root in a DDD-based EF Core model?
25. How do you optimize EF Core queries for high-traffic read workloads?
26. What strategies do you use to reduce N+1 queries in EF Core?
27. How do compiled queries improve performance in EF Core?
28. What problems arise from using `AutoInclude` and when should it be avoided?
29. How do you implement soft delete in EF Core?
30. What is the difference between Table-per-Hierarchy, Table-per-Type and Table-per-Concrete-Type inheritance?
31. How do you implement optimistic concurrency tokens in EF Core?
32. How does connection pooling behave in EF Core?
33. How do you use interceptors to customize EF Core behavior?
34. What is the difference between tracking graphs and attaching detached entities?
35. Do you use the Repository pattern with EF Core?
36. What is the UnitOfWork pattern and how is it related to EF Core?
37. How to start a transaction in EF Core?
38. How do you handle transactional boundaries across multiple DbContexts?
39. How do you implement read/write separation using multiple DbContexts (CQRS)?
40. How do you apply migrations in production?
