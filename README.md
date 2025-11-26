# Introduction to the Bug Hunt Challenge  -  Assignment 

My City is a SwiftUI travel guide App.  The App shows attractions in a number of European cities - ```Paris, Rome, Barcelona, and Berlin```.

Users can:

- Browse a List of attractions
- Filter them by category - ```Food, Museum, Sports, and Historical```,
- Search by name or city,
- Tap the heart to mark favorites and then filter to show favorites only.

The view is driven by SwiftUI state such that when any state value changes, SwiftUI will the view hierarchy and the UI updates.



## Screenshots of the output of the program:
<img width="936" height="1756" alt="image" src="https://github.com/user-attachments/assets/b73a585e-c3a0-4e07-bb5c-821f50a7f1c2" />

Figure 1: On View on Loading the App



<img width="770" height="1722" alt="image" src="https://github.com/user-attachments/assets/6b80796e-46c7-4581-bd11-19ddf7dacc98" />

Figure 2: Search Results View



## Bug Identification

In the *My City* project the following *logic and UI bugs* were found:

1. **Missing return type on `filteredAttractions`** 

```
Origianal code:

// ORIGINAL (compile error)  // Highlighted by the IDE
private var filteredAttractions { 
    var results = attractions
    // ... filtering logic ...
    return results
}
```

Corrected code - with return type


```
// Corrected
private var filteredAttractions: [Attraction] {
    var results = attractions
    // ... filtering logic ...
    return results
}
```

 
References - https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties - under the section on Properties 


 - filteredAttractions is a computed property and therefore it doesn’t store a value, it computes it every time from other properties.
 - In Swift, a property declaration must include a type annotation (or a type that the compiler can infer directly).
 - For computed properties you typically declare the type explicitly. 


This caused a compile-time error because Swift requires a return type for every computed property.

2. **Category filter  - != Vs ==**
   
The above is found in the filteredAttractions
   

   ```
   // ORIGINAL – inverted logic
        if let category = selectedCategory {
            results = results.filter { $0.category != category }
        }
   ```

   
When the user selects "Food", it removed the food attractions instead of keeping them. So we modify the code to keep them as follows:


```
// Corrected codes -  keep only matching category
if let category = selectedCategory {
    results = results.filter { $0.category == category }
}
```


References: https://developer.apple.com/documentation/swift/array

- filter() returns a new array containing only the elements that satisfy the predicate.
- Hence the correct predicate must be “category equals selectedCategory”, so the correct operator is ==, not !=.



3. **Favorites filter  - here we have inverted logic**

Inside the filteredAttractions:

```
// ORIGINAL - The code hides favorites when showFavorites is true
if showFavorites {
    results = results.filter { !favoriteIds.contains($0.id) }
}
```
The code turning on the favorites filter ends up removing the favourites from the list.

```
// Corrected - This will show only favorites when showFavorites is true
if showFavorites {
    results = results.filter { favoriteIds.contains($0.id) }
}
```



- The correct predicate is “ID is contained in the set of favorite IDs”.
- Sets are collections of unique elements with fast membership tests.


```
// Corrected  -  filled heart means  that the “favorites filter is ON”
.toolbar {
    ToolbarItem(placement: .navigationBarTrailing) {
        Button(action: {
            showFavorites.toggle()
        }) {
            Image(systemName: showFavorites ? "heart.fill" : "heart")
                .foregroundColor(.red)
        }
        .accessibilityLabel(showFavorites ? "Show all attractions" : "Show favourites only")
    }
}
```



```
// Corrected  -  We only show the favorites when showFavorites is true
if showFavorites {
    results = results.filter { favoriteIds.contains($0.id) }
}
```


References: https://developer.apple.com/documentation/swift/set


 - favoriteIds is a Set<UUID> storing the IDs of favorited attractions (Apple Developer website).
 - Sets are collections of unique elements with fast membership tests.
 - The correct predicate is “ID is contained in the set of favorite IDs”


Therefore, Enabling the “favorites” filter. 


4. **Togggling favorites by use of Set.insert/ set.remove**

This is found in the List where each row passes a closure to AttractionRowView.

```
// ORIGINAL -  only add favorites, never remove it 
List(filteredAttractions) { attraction in
    AttractionRowView(
        attraction: attraction,
        isFavorite: favoriteIds.contains(attraction.id),
        toggleFavorite: {
            favoriteIds.insert(attraction.id)
        }
    )
}
```

This lives in the List where each row passes a closure to AttractionRowView.



```
// corrected  - true will toggle on a Set
List(filteredAttractions) { attraction in
    AttractionRowView(
        attraction: attraction,
        isFavorite: favoriteIds.contains(attraction.id),
        toggleFavorite: {
            if favoriteIds.contains(attraction.id) {
                favoriteIds.remove(attraction.id)
            } else {
                favoriteIds.insert(attraction.id)
            }
        }
    )
}

```


  - This is a standard pattern used for toggling membership in a Set.

Tapping the heart several times only ever added IDs to the set; there was no way to toggle and remove it.



Reference: https://developer.apple.com/documentation/swift/set

 - Set.insert() adds an element if it isn’t already present.
 - Set.remove(_:) removes an element from the set.
 - This is a standard pattern for toggling membership in a Set.



5. **Toolbar heart Icon - need the mapping @state to UI**

   When showFavorites is set to true, the UI will show an empty heart. 

```
// ORIGINAL - the  icon meaning is reversed
.toolbar {
    ToolbarItem(placement: .navigationBarTrailing) {
        Button(action: {
            showFavorites.toggle()
        }) {
            Image(systemName: showFavorites ? "heart" : "heart.fill")
                .foregroundColor(.red)
        }
    }
}
```


References: https://developer.apple.com/documentation/swiftui/state


```
// Corrected  -  filled heart means “favorites filter is ON”
.toolbar {
    ToolbarItem(placement: .navigationBarTrailing) {
        Button(action: {
            showFavorites.toggle()
        }) {
            Image(systemName: showFavorites ? "heart.fill" : "heart")
                .foregroundColor(.red)
        }
        .accessibilityLabel(showFavorites ? "Show all attractions" : "Show favourites only")
    }
}

```

 - showFavorites is marked with @State, so changing it automatically will recompute the view and updates the icon.
 - SwiftUI’s state system is built so that “the UI is a function of state”; the icon is just a projection of the showFavorites Boolean (Apple Developer).




6. Rating and stars – in the AttractionRowView

    - In this section the app displays rating stars and a numeric rating for each attraction.
    - The problem with this is that: Every row showed five filled stars, even for poor ratings.
    - The numeric rating was also doubled (rating * 2), and hence the real-world 0–5 ratings appeared as 0–10.


// ORIGINAL AttractionRowView code
struct AttractionRowView: View {
    let attraction: Attraction
    let isFavorite: Bool
    let toggleFavorite: () -> Void

    var body: some View {
        HStack {
            Text(attraction.category.rawValue)
                .font(.system(size: 40))

            VStack(alignment: .leading, spacing: 4) {
                Text(attraction.name)
                    .font(.headline)

                Text(attraction.description)
                    .font(.subheadline)
                    .foregroundColor(.gray)

                HStack {
                    HStack(spacing: 2) {
                        // BUG: always shows 5 filled stars,
                        // regardless of rating
                        ForEach(0..<5) { _ in
                            Image(systemName: "star.fill")
                                .foregroundColor(.yellow)
                                .font(.system(size: 12))
                        }
                    }

                    // BUG: multiplies rating by 2, so a 4.8 rating shows 9.6
                    Text(String(format: "%.1f", attraction.rating * 2))
                        .font(.caption)
                        .foregroundColor(.gray)

                    Spacer()

                    Text(attraction.isOpen ? "Open" : "Closed")
                        .font(.caption)
                        .padding(.horizontal, 8)
                        .padding(.vertical, 4)
                        .background(attraction.isOpen ? Color.green.opacity(0.2) :
                                      Color.red.opacity(0.2))
                        .cornerRadius(4)
                }
            }

            Spacer()

            Button(action: toggleFavorite) {
                Image(systemName: isFavorite ? "heart.fill" : "heart")
                    .foregroundColor(isFavorite ? .red : .gray)
                    .font(.system(size: 20))
            }
            .buttonStyle(BorderlessButtonStyle())
        }
        .padding(.vertical, 8)
    }
}



 - ForEach in SwiftUI builds a view for each item in a collection; here we use it to draw five star icons, choosing filled vs empty based on the index.
 - List is the SwiftUI container that shows one AttractionRowView per attraction.
 - The rating logic is just pure Swift math, but wrapped in computed properties (again, values derived from other stored data rather than stored themselves).

```
   // Corrected code AttractionRowView
struct AttractionRowView: View {
    let attraction: Attraction
    let isFavorite: Bool
    let toggleFavorite: () -> Void

    // Clamp rating into 0...5
    private var clampedRating: Double {
        min(max(attraction.rating, 0), 5)
    }

    // Rounded number of filled stars
    private var filledStars: Int {
        Int(clampedRating.rounded())
    }

    var body: some View {
        HStack {
            Text(attraction.category.rawValue)
                .font(.system(size: 40))

            VStack(alignment: .leading, spacing: 4) {
                Text(attraction.name)
                    .font(.headline)

                Text(attraction.description)
                    .font(.subheadline)
                    .foregroundColor(.gray)

                HStack {
                    // Stars now reflect the rating
                    HStack(spacing: 2) {
                        ForEach(0..<5) { index in
                            Image(systemName: index < filledStars ? "star.fill" : "star")
                                .foregroundColor(.yellow)
                                .font(.system(size: 12))
                        }
                    }

                    // Display the real (clamped) rating
                    Text(String(format: "%.1f", clampedRating))
                        .font(.caption)
                        .foregroundColor(.gray)

                    Spacer()

                    Text(attraction.isOpen ? "Open" : "Closed")
                        .font(.caption)
                        .padding(.horizontal, 8)
                        .padding(.vertical, 4)
                        .background(attraction.isOpen ? Color.green.opacity(0.2) :
                                      Color.red.opacity(0.2))
                        .cornerRadius(4)
                }
            }

            Spacer()

            Button(action: toggleFavorite) {
                Image(systemName: isFavorite ? "heart.fill" : "heart")
                    .foregroundColor(isFavorite ? .red : .gray)
                    .font(.system(size: 20))
            }
            .buttonStyle(BorderlessButtonStyle())
        }
        .padding(.vertical, 8)
    }
}

```
   
\ **Minor UX bug in favorites button**
   The toolbar heart icon showed `heart` when favorites were active and `heart.fill` when inactive, reversing the usual convention and confusing the meaning of the button.

Together these issues affected compilation, filtering logic, and how trustworthy the UI felt.





  ```swift
  print("After category filter:", results.count)
  print("Favorites only:", showFavorites, "current favorites:", favoriteIds.count)
  ```

  This confirmed that the wrong items were being removed when a category or favorites filter was enabled.
* **Quick inspections in the debugger (`po` in LLDB)** to verify the values of `selectedCategory`, `showFavorites`, and `favoriteIds` at runtime.
* **UI inspection in the preview** to verify star counts and rating labels visually after changes to `AttractionRowView`.

Using these tools together gave me both *what* was wrong (compiler and preview) and *why* it was wrong (logging / inspection).

---

## 3. Fix Implementation

I then implemented and tested a set of targeted fixes:

1. **Give `filteredAttractions` an explicit type**

   ```swift
   private var filteredAttractions: [Attraction] {
       var results = attractions
       ...
       return results
   }
   ```

   This resolved the compile error and clearly communicates the intention of the property.

2. **Correct the category filter logic**

   ```swift
   if let category = selectedCategory {
       results = results.filter { $0.category == category }
   }
   ```

   Now selecting “Food” correctly shows only food attractions, etc.

3. **Correct the favorites filter logic**

   ```swift
   if showFavorites {
       results = results.filter { favoriteIds.contains($0.id) }
   }
   ```

   With this change, turning on the favorites filter restricts the list to items whose IDs are in the `favoriteIds` set.

4. **Implement a true toggle for favorites**

   ```swift
   AttractionRowView(
       attraction: attraction,
       isFavorite: favoriteIds.contains(attraction.id),
       toggleFavorite: {
           if favoriteIds.contains(attraction.id) {
               favoriteIds.remove(attraction.id)
           } else {
               favoriteIds.insert(attraction.id)
           }
       }
   )
   ```

   Tapping the heart now adds a favorite if it is not present, or removes it if it already is.

5. **Fix rating display and star rendering (in `AttractionRowView`)**

   * Clamp the rating and compute how many stars to fill:

     ```swift
     private var clampedRating: Double { min(max(attraction.rating, 0), 5) }
     private var filledStars: Int { Int(clampedRating.rounded()) }
     ```

   * Use that to draw the stars and label:

     ```swift
     ForEach(0..<5) { index in
         Image(systemName: index < filledStars ? "star.fill" : "star")
     }
     Text(String(format: "%.1f", clampedRating))
     ```

   This keeps the numeric rating in the 0–5 range and aligns the icon display with the value.

6. **Align the toolbar heart icon with the filter state**

   ```swift
   Image(systemName: showFavorites ? "heart.fill" : "heart")
       .foregroundColor(.red)
   ```

   Now `heart.fill` clearly means “favorites filter active”.



## 4. Reflection

Working on this debugging task reinforced the following concpets: 

* **Start with compiler errors, then move to behavior.** Fixing the type error on `filteredAttractions` first unblocked the project and made it easier to reason about runtime behavior.
  
* **Logic bugs often hide in “small” operators.** Swapping `!=` for `==` and removing the `!` were single-character changes, but they completely flipped the meaning of the filters. Reading code in plain English (“keep only items where category equals the selected category”) helped me spot these mistakes.

  
* **Importance of Testing with real usage scenarios.** Simply looking at the code would not have revealed that favorites could never be removed. Clicking through the UI like an actual user quickly exposed the problem.

  
* **Multiple feedback channels make debugging faster.** Combining preview, logging, and occasional debugger inspection gave me both a high-level feel for the UI and low-level visibility into data flow.
  
* **Clean, explicit code reduces future bugs.** Adding an explicit type to `filteredAttractions` and encapsulating rating clamping in helper properties make the intent clearer for anyone including for my future use  - reading or modifying this code.

In general, the exercise showed me how small logic errors propagate into confusing UX, and how a systematic debugging approach - observe, hypothesize, instrument, fix, retest-leads to clean, reliable SwiftUI code.
