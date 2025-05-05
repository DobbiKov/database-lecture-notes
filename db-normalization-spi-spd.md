Alright class, let's dive into a really important topic when we're designing our database schemas: what happens when we decide to break down (or decompose) a large table into smaller ones? While this process, called **normalisation**, is often a good idea because it helps us avoid problems like repeating the same information everywhere and makes updates easier, we have to be super careful about two potential issues. These are what you've referred to as "SPI" and "SPD," which correspond to **"Sans Perte d’information" (Without Loss of Information)** and **"Perte de Dépendance" (Loss of Dependency)** in French, based on our sources.

Think of it like taking a complex document and splitting it into several smaller notes. You need to make sure you haven't lost any important details in the process (that's related to "Sans Perte d'information") and that the original rules or connections between different pieces of information are still clear and enforceable across your notes (that's related to "Perte de Dépendance").

Let's break them down, just like we might break down a schema!

**1. Décomposition Sans Perte d’information (Without Loss of Information - SPI)**

*   **What it is:** A decomposition of a relation schema (like a table structure) is **without loss of information** if, when you take any valid set of data (an instance) for the original table, break it down into instances for the new, smaller tables, and then put those smaller instances back together using a join operation, you get *exactly* the original set of data back. You don't lose any original rows, and crucially, you don't gain any *new*, incorrect rows (sometimes called "spurious" tuples).
*   **Why it matters:** If a decomposition *has* loss of information, joining the decomposed tables can create combinations of data that didn't exist in the original table. This means querying the joined tables gives you a result that isn't true based on the original data, which is obviously a big problem for data integrity and accuracy. The sources state that *any* update or query possible with the original schema must also be possible with the decomposed schema.
*   **Simple Example from the Sources:** Let's look at R(Titre, Nom-Ciné, Tél) without any initial dependencies. Imagine you have this data:
    ```
    r:
    Titre    | Nom-Ciné | Tél
    ----------|----------|------
    Speed 2  | Français | ..1187
    Speed 2  | Français | ..1188
    Marion   | Français | ..1187
    ```
    Now, you decide to decompose it into R1(Titre, Nom-Ciné) and R2(Nom-Ciné, Tél). The decomposed instances would look like this:
    ```
    r1:                    r2:
    Titre    | Nom-Ciné      Nom-Ciné | Tél
    ----------|----------      ----------|------
    Speed 2  | Français      Français | ..1187
    Marion   | Français      Français | ..1188
    ```
    If you join r1 and r2 back together on the common attribute `Nom-Ciné`, what do you get?
    ```
    r1 ⋈ r2:
    Titre    | Nom-Ciné | Tél
    ----------|----------|------
    Speed 2  | Français | ..1187  (This was in the original r)
    Speed 2  | Français | ..1188  (This was in the original r)
    Marion   | Français | ..1187  (This was in the original r)
    **Marion   | Français | ..1188**  (**This row is NEW! It wasn't in the original data!**)
    ```
    This decomposition suffers from **loss of information** because the join introduced a tuple that wasn't originally present. The source asks, "where to call for info about films?" highlighting the ambiguity introduced.
*   **How to achieve it (linked to dependencies):** The sources provide a crucial condition for a decomposition of R(XYZ) into R1(XY) and R2(XZ) to be **without loss of information**: it happens if and only if either the functional dependency X → Y is implied by the original set of dependencies F, OR the functional dependency X → Z is implied by F. This shows the critical relationship between functional dependencies and achieving lossless decomposition.

**2. Perte de Dépendance (Loss of Dependency - SPD)**

*   **What it is:** A decomposition suffers from **loss of dependency** if some functional dependency from the original schema's set of dependencies (F) that was true for the original table is *not* logically implied by the set of projected dependencies on the decomposed tables (the Fi sets). Even if each smaller table satisfies its own dependencies, the *combination* might allow data that violates an original rule.
*   **Why it matters:** Functional dependencies represent constraints that data must satisfy. If these constraints are lost in the decomposition, you can perform updates (insertions, modifications) on the smaller tables that result in data in the decomposed schema which, if viewed or joined back together, would violate the original constraints. This compromises the integrity of your data.
*   **Simple Example from the Sources:** Consider the table R(Titre Acteur Nom-Ciné) with the functional dependencies F = {Nom-Ciné → Titre, Titre Acteur → Nom-Ciné}.
    *   `Nom-Ciné → Titre`: A cinema name determines the title of the film being shown (at a specific time/context, though the example is simplified).
    *   `Titre Acteur → Nom-Ciné`: A film title and an actor appearing in it might determine the cinema where a related event (like a conference) takes place.
    Let's decompose this into R1(Nom-Ciné, Titre) with the projected dependency Nom-Ciné → Titre, and R2(Nom-Ciné, Acteur) (which the source implies has no relevant projected dependency from F for this example).
    Suppose you have an instance satisfying F:
    ```
    r:
    Titre    | Acteur  | Nom-Ciné
    ----------|---------|----------
    Speed2   | Bullock | Français
    Speed2   | Patric  | Trianon
    ```
    The decomposed instances are:
    ```
    r1:                    r2:
    Nom-Ciné | Titre         Nom-Ciné | Acteur
    ----------|----------      ----------|---------
    Français | Speed2        Français | Bullock
    Trianon  | Speed2        Trianon  | Patric
    ```
    Now, imagine you insert the tuple (Trianon, Bullock) into R2. R2 now looks like:
    ```
    r2_new:
    Nom-Ciné | Acteur
    ----------|---------
    Français | Bullock
    Trianon  | Patric
    **Trianon  | Bullock**
    ```
    There is no functional dependency *within* R2 that would prevent this insertion.
    However, if you join r1 and r2_new:
    ```
    r1 ⋈ r2_new:
    Titre    | Acteur  | Nom-Ciné
    ----------|---------|----------
    Speed2   | Bullock | Français
    Speed2   | Patric  | Trianon
    **Speed2   | Bullock | Trianon** (**This tuple violates Nom-Ciné → Titre!**)
    ```
    In the joined result, 'Trianon' is now associated with 'Speed2' *and* potentially other titles if they existed in r1 associated with 'Trianon'. The original dependency `Nom-Ciné → Titre` which requires 'Trianon' to determine *one* unique `Titre` (like 'Speed2' based on the original r1 data) is violated by the appearance of the new tuple (`Speed2, Bullock, Trianon`). This shows **loss of dependency** because the constraint `Nom-Ciné → Titre` was not enforceable simply by looking at the constraints on R1 and R2 independently during the update.

In summary, when decomposing schemas as part of normalisation, we aim to eliminate redundancy and improve update behaviour. However, we must always strive for **lossless decomposition** (SPI) so that the original data can be perfectly reconstructed. We also need to be aware of **loss of dependency** (SPD), as it means original constraints are not fully preserved, potentially leading to data integrity issues during updates, even if the decomposition itself is lossless. It's a balance to strike in database design!
