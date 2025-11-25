# Introduction to the Bug Hunt Challenge - Assignment 

My City is a SwiftUI travel guide App.  The App shows attractions in a number of European cities - ```Paris, Rome, Barcelona, and Berlin```.

Users can:

- Browse a List of attractions,
- Filter them by category - ```Food, Museum, Sports, and Historical```,
- Search by name or city,
- Tap the heart to mark favorites and then filter to show favorites only.

The view is driven by SwiftUI state such that when any state value changes, SwiftUI will the view hierarchy and the UI updates.


## Screenshots of the output of the program:
<img width="936" height="1756" alt="image" src="https://github.com/user-attachments/assets/b73a585e-c3a0-4e07-bb5c-821f50a7f1c2" />

Figure 1: On View on Loading the App


<img width="770" height="1722" alt="image" src="https://github.com/user-attachments/assets/6b80796e-46c7-4581-bd11-19ddf7dacc98" />

Figure 2: Search Results View



## 1. Bug Identification

In the *My City* project the following *logic and UI bugs* were found:

1. **Missing return type on `filteredAttractions`** -

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
// FIXED
private var filteredAttractions: [Attraction] {
    var results = attractions
    // ... filtering logic ...
    return results
}

```

 
References - https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties - under the section on Properties 


filteredAttractions is a computed property and therefore it doesn’t store a value, it computes it every time from other properties. 
In Swift, a property declaration must include a type annotation (or a type that the compiler can infer directly). 
For computed properties you typically declare the type explicitly. 



   ```swift
   var filteredAttractions {
   ```

   ``` Testing out things ```

   This caused a compile-time error because Swift requires a return type for every computed property.

1. **Category filter inverted**

   ```swift
   results = results.filter { $0.category != category }
   ```

   When a category was selected, the app removed those attractions instead of keeping them, so the filter behaved opposite to what the user expects.

2. **Favorites filter inverted**

   ```swift
   results = results.filter { !favoriteIds.contains($0.id) }
   ```

   Enabling the “favorites” filter actually hid favorites and showed everything else.

3. **Favorites could not be removed**

   ```swift
   toggleFavorite: {
       favoriteIds.insert(attraction.id)
   }
   ```

   Tapping the heart only ever added IDs to the set; there was no way to toggle a favorite off.

4. **Rating display and stars inconsistent (in `AttractionRowView`)**

   ```swift
   Text(String(format: "%.1f", attraction.rating * 2))
   ```

   The label doubled the rating (displaying up to 10.0), and the star icons were always filled, regardless of the underlying rating. This made the UI misleading.

5. **Minor UX bug in favorites button**
   The toolbar heart icon showed `heart` when favorites were active and `heart.fill` when inactive, reversing the usual convention and confusing the meaning of the button.

Together these issues affected compilation, filtering logic, and how trustworthy the UI felt.

---

## 2. Debugging Tools

To track these down I used several debugging tools and techniques:

* **Compiler errors and Xcode warnings** for the missing type on `filteredAttractions`. The red error pointed directly to the property and made it clear I needed a return type.
* **SwiftUI Preview / live preview** to interactively run the app and observe behavior as I changed filters and toggled favorites without doing a full simulator run each time.
* **Print logging** inside `filteredAttractions` to inspect the count and contents of `results` after each filter step:

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

After each change I re-ran the SwiftUI Preview and manually tested: changing categories, searching, toggling favorites on and off, and checking that the ratings and stars look correct. The app now compiles cleanly and the filters behave as expected.

---

## 4. Reflection

Working through this debugging task reinforced a few key lessons:

* **Start with compiler errors, then move to behavior.** Fixing the type error on `filteredAttractions` first unblocked the project and made it easier to reason about runtime behavior.
* **Logic bugs often hide in “small” operators.** Swapping `!=` for `==` and removing a stray `!` were single-character changes, but they completely flipped the meaning of both filters. Reading code in plain English (“keep only items where category equals the selected category”) helped me spot these mistakes.
* **Testing with real usage scenarios is essential.** Simply looking at the code would not have revealed that favorites could never be removed. Clicking through the UI like an actual user quickly exposed the problem.
* **Multiple feedback channels make debugging faster.** Combining preview, logging, and occasional debugger inspection gave me both a high-level feel for the UI and low-level visibility into data flow.
* **Clean, explicit code reduces future bugs.** Adding an explicit type to `filteredAttractions` and encapsulating rating clamping in helper properties make the intent clearer for anyone (including my future self) reading or modifying this code.

Overall, the exercise showed me how small logic errors propagate into confusing UX, and how a systematic debugging approach—observe, hypothesize, instrument, fix, retest—leads to clean, reliable SwiftUI code.
