---
title: Sorting algorithms in Rust
date: 2020-05-28
spoiler: Quick sort, Bubble sort, Merge sort
language: en
tags:
  - programming
  - rust
  - algorithms
  - post
---

_Epistemic status: Still learning_

While reading "De-Coding The Technical Interview Process" by Emma Bostian (@EmmaBostian) which has great examples of sorting algorithms in Javascript, I wondered whether there are any good examples in Rust out there.

I couldn't find an article which has more than one algorithm, so I took each example, tweaked it a little bit and put together in one collection. The implementations are pretty different from Javascript versions in the book. I will add explanations to this post later.

Here are Quick sort, Bubble sort and Merge sort implemented in Rust.

## Bubble sort

This is the easiest one.

```rust
fn sort(array: &mut Vec<i32>) {
  for i in 0..array.len() {
    for j in 0..array.len() - i - 1 {
      if array[j + 1] < array[j] {
        // let tmp = array[j];
        // array[j] = array[j + 1];
        // array[j + 1] = tmp;
        array.swap(j, j + 1);
      }
    }
  }
}
```

## Merge sort

This one is best done with two functions: `sort` and `merge`.

```rust
fn sort(array: &mut [i32]) {
  let middle = array.len() / 2;
  if array.len() < 2 {
    return; // No need to sort vectors with one element
  }

  let mut sorted = array.to_vec();

  sort(&mut array[..middle]);
  sort(&mut array[middle..]);

  merge(&array[..middle], &array[middle..], &mut sorted);

  array.copy_from_slice(&sorted); // Copy the sorted result into original vector
}

fn merge(l_arr: &[i32], r_arr: &[i32], sorted: &mut [i32]) {
  // Current loop position in left half, right half, and sorted vector
  let (mut left, mut right, mut i) = (0, 0, 0);

  while left < l_arr.len() && right < r_arr.len() {
    if l_arr[left] <= r_arr[right] {
      sorted[i] = l_arr[left];
      i += 1;
      left += 1;
    } else {
      sorted[i] = r_arr[right];
      i += 1;
      right += 1;
    }
  }

  if left < l_arr.len() {
    // If there is anything left in the left half append it after sorted members
    sorted[i..].copy_from_slice(&l_arr[left..]);
  }

  if right < r_arr.len() {
    // If there is anything left in the right half append it after sorted members
    sorted[i..].copy_from_slice(&r_arr[right..]);
  }
}
```

## Quick sort

This one is the trickiest, since [partitioning](https://www.geeksforgeeks.org/quick-sort/) is done in-place. Also, since there is no way to set default parameter values in Rust, we need a third, "entry" function, so that we don't have to explicitly specify that we need to sort the whole array from 0 to last index.

```rust
fn sort(array: &mut [i32]) {
  let start = 0;
  let end = array.len() - 1;
  quick_sort_partition(array, start, end as isize);
}

fn quick_sort_partition(array: &mut [i32], start: isize, end: isize) {
  if start < end && end - start >= 1 {
    let pivot = partition(array, start as isize, end as isize);
    quick_sort_partition(array, start, pivot - 1);
    quick_sort_partition(array, pivot + 1, end);
  }
}

fn partition(array: &mut [i32], l: isize, h: isize) -> isize {
  let pivot = array[h as usize];
  let mut i = l - 1; // Index of the smaller element

  for j in l..h {
    if array[j as usize] <= pivot {
      i = i + 1;
      array.swap(i as usize, j as usize);
    }
  }

  array.swap((i + 1) as usize, h as usize);

  i + 1
}
```

You can also find them on Github: [rust-sorting-algorithms](https://github.com/jlkiri/rust-sorting-algorithms)
