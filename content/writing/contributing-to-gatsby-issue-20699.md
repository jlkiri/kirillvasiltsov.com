---
title: "Contributing to Gatsby - Issue #20699"
date: "2020-06-26"
spoiler: Do you know how long can your directory names be?
language: en
tags:
  - OSS
  - programming
  - post
---

[Last time](https://www.kirillvasiltsov.com/writing/contributing-to-gatsby-issue-21311/) I wrote about fixing `gatsby-transformer-remark`, which failed to properly count words (and time-to-read) in Chinese and Japanese languages.

Today I'm going to tell you about another contribution to [Gatsby](https://www.gatsbyjs.org/) that I made earlier this year.

## Background

First of all, here's [the issue](https://github.com/gatsbyjs/gatsby/issues/20699) that is the topic of this post. It describes a possible build error when directories/filenames built by Gatsby are **too long**. Additionally, there isn't a good error message and all a user would see is `ENAMETOOLONG`.

Apart from being a crash, this is bad for two reasons:

a) This can happen by using Gatsby the right, documented way  
b) This is caused by files/directories that are not supposed to be seen/touched by a user and no obvious workaround is possible

Why does this happen?

Gatsby allows you to [create pages programmatically](https://www.gatsbyjs.org/docs/creating-slugs-for-pages/). This means that based on your local or fetched data, you can choose the **slug** for your page: `example/of/programmatically-created-slug`. However, there are no rules about what slugs should look like. When you run `gatsby build`, Gatsby actually creates folders with names that correspond to segments of the slug you created (and puts associated data and other interesting stuff inside). This implies that if you have a slug with a segment that is 1000 characters long, **a directory with a name 1000 characters long will be created**.

The problem is, on most operating systems you cannot create directories that long!

So, the possible solution is to either crash with a meaningful message, or automatically truncate paths that are too long and warn about the length. This is what I did in [the pull request](https://github.com/gatsbyjs/gatsby/pull/21518).

## Fixing the bug

Under the hood, Gatsby uses [Redux](https://redux.js.org/) to manage state during build. Most of Gatsby's API are [**actions**](https://www.gatsbyjs.org/docs/actions/) like `createNodeField` which you can call by hooking into different stages of build, e.g. `onCreateNode`. The code that handles public actions is inside `/src/redux/actions/public.js`. Inside of this file, there is a `createPage` action that you call when you want to programmatically create a page. And here is how a page is represented internally:

```javascript
const internalPage: Page = {
  internalComponentName,
  path: page.path,
  matchPath: page.matchPath,
  component: page.component,
  componentChunkName: generateComponentChunkName(page.component),
  isCreatedByStatefulCreatePages:
    actionOptions && actionOptions.traceId === `initial-createPagesStatefully`,
  context: page.context || {},
  updatedAt: Date.now(),
};
```

Of these properties, we are concerted with `path`. We can check it for segments that are too long. So, before creating the object above, let us do something like this:

```javascript
const invalidPathSegments = tooLongSegmentsInPath(page.path);

if (invalidPathSegments.length > 0) {
  const truncatedPath = truncatePath(page.path);
  page.path = truncatedPath;
}
```

`tooLongSegmentsInPath` is where it gets interesting. First of all, we simply split the path by `/`. Then we need to check whether each segment is too long.

```javascript
export const tooLongSegmentsInPath = (path: string): Array<string> => {
  const invalidFilenames: Array<string> = [];
  for (const segment of path.split(`/`)) {
    if (isNameTooLong(segment)) {
      invalidFilenames.push(segment);
    }
  }
  return invalidFilenames;
};
```

But what is _too long_? It [seems that most OS have a filename limit of 255 bytes](https://serverfault.com/questions/9546/filename-length-limits-on-linux/9548#9548). However, I did some research and found out that some systems allow 255 **characters**! For example, both MacOS (APFS) and Windows (NTFS) allow up to 255 chars, regardless of how many bytes is in one character. For example, a Latin character `a` is 1 byte, but a Japanese character `あ` is all 4! On macOS and Windows, I'm allowed to create a folder which name is 255 characters `あ` (=1020 bytes). Anyway, it seems like there aren't systems that allow less than 255 bytes, so we need to put that in code.

To find out which OS is used to `build`, we use a very handy Node.js `process.platform` property:

```javascript
const isMacOs = process.platform === `darwin`; // Yes, it is "darwin"
const isWindows = process.platform === `win32`;
```

And here is the `isNameTooLong` function:

```javascript
const MAX_PATH_SEGMENT_CHARS = 255;
const MAX_PATH_SEGMENT_BYTES = 255;

const isNameTooLong = (segment: string): boolean =>
  isMacOs || isWindows
    ? segment.length > MAX_PATH_SEGMENT_CHARS // MacOS (APFS) and Windows (NTFS) filename length limit (255 chars)
    : Buffer.from(segment).length > MAX_PATH_SEGMENT_BYTES; // Other (255 bytes)
```

A good way to truncate a path is to cut it at around 50 chars and to hash the rest with a function that hashes strings to obtain a unique value. This way we don't accidentally get identical truncated paths. Luckily, Gatsby already has a hashing function `createContentDigest`, so I just used it.

```javascript
export const truncatePath = (path: string): string =>
  path.replace(pathSegmentRe, (match) => {
    if (isNameTooLong(match)) {
      return (
        match.slice(0, SLICING_INDEX) +
        createContentDigest(match.slice(SLICING_INDEX))
      );
    }
    return match;
  });
```

Finally, we want to truncate the path only during `development` and show a warning. This is because the truncation is automatic and the resulting hash is unpredictable to the user. This can be easily done by looking at `NODE_ENV` environment variable. In `production` paths that are too long are still expected to cause crash but with a meaningful error.

As always, new functions need to be unit tested, so I wrote a couple of tests for the functions above, which you can see in the PR. And that's it!

## Next time

I plan to write more about contributions in the near future. In the meantime, if you liked the article, [follow me on Twitter](https://twitter.com/virtualkirill) for more updates! Also, [check out the first post](https://www.kirillvasiltsov.com/writing/contributing-to-gatsby-issue-21311/) about my contribution to Gatsby if you haven't yet.
