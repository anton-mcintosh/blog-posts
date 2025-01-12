---
title: How My Blog Pipeline Works
tags:
  - markdown
  - webdev
  - html
  - Go
  - github
  - javascript
nutshell: Blog built from scratch. Read to learn how it works.
topic: Tech
readtime: 12 min read
---
One of my goals for 2025 is to write more. Ideally I'd like to be writing something every day. Having a blog sounded like something that would give me some accountability to write thoughtfully since other people will (maybe) read it. Yes, I'm being quite presumptive to assume that someone might want to read what I write in a format that has seemingly fallen out of favor. 

My other goal for 2025 is to get a job using my shiny new Computer Science degree. Unfortunately I didn't leave Western Washington University's CS program with many projects that are interesting to prospective employers. So I decided to endeavor to build a blog, from scratch, to learn some new skills and have something meaningful to show for the effort.

<!-- summary -->
## Motivation

I've been keeping a daily journal, as well as general note taking, using [Obsidian](https://obsidian.md/) for some time now. There is a lot for me to love about Obsidian. For one thing, I can use Vim key bindings, to which I've grown accustomed through my use of [Neovim](https://neovim.io/) as my main text editor. Additionally, Obsidian uses [Markdown](https://www.markdownguide.org/cheat-sheet/) which makes formatting plaintext a breeze.

So I wondered, could there be a way to write my blog posts in Obsidian, and then automate the process of posting those to a blog on my website? I did a little digging to see if it had been done already and found [Network Chuck's](https://blog.networkchuck.com/) blog in which he had done something similar. Chuck used [Hugo](https://gohugo.io/) which is a static site generator that seems like it made the task of creating an automated blog pipeline pretty simple. Simple isn't what I was after, though, since i'm trying to learn some nuts and bolts of web development. So I endeavored to make things harder than they need to be and build my blog API from scratch using Go.

### A quick word about the cloud.
When I first decided I wanted to build a website, I considered the possibilities of how I wanted to host it. Of course there are options such as Wix or Squarespace, but as a recent CS grad, that felt somewhat like cheating. Would my website look much better if I used those solutions? 100% it would. I do **NOT** have an eye for design, and I know this. The fact that you're here reading this tells me that you know this about me as well.

I decided that I wanted roll my own webserver. Now, I have a homelab server that could easily run an Nginx container and go from there. The idea of giving the whole world access to the inside of my house, albeit digitally and securely, made me queesy.

In college I took a class on cloud security that used AWS. Lots of companies use AWS so this was my first logical choice. However, if you're not VERY careful, you can wake up to some big charges. It is just not super obvious for someone learning how to stay out of trouble with AWS. It was always stressed in class as some students had forgotten to disable an EC2 instance or some other product they were playing with and ended up incurring thousands of dollars in charges that were difficult, if at all possible, to reverse.

After doing some research I ended up launching a Digital Ocean Droplet. This runs me around $6 per month and if I need to scale it for additional bandwidth or storage, their pricing is very transparent. Basically this is just a simple linux virtual machine that i'm paying for. At the end of the day, that's all the "cloud" really is: virtual computers hosted by another company that you pay to use. Of course there are extra products and services that they tack on (that cost more) and make things more complex, but I didn't need all that.

After running this server for a few months, I definitely recommend Digital Ocean if you're looking to build personal projects on a budget.
## Backend - Go and Gin

#### Why Go?
The experienced reader might wonder why I didn't decide to use Node or Python for my blog's API. Primarily I didn't want to use Node or Python because I've already built APIs with those for other projects and I wanted to learn Go. I learn a lot about how something works by experiencing it from a few different angles. Building my backend using Go helped me solidify my understanding of how an API works under the hood.

Also, to be honest, i'm not a big fan of Python. The main languages I learned while getting my CS degree were Java and C. After writing so much code in those languages, my experience having to build a project in Python was frustrating and I never fully recovered from it. Go strikes a nice balance  between the C world and the Python world. Go is statically typed, pretty fast and has a syntax closer to my comfort zone. The main downside of using Go would end up being the fact that it just isn't as widely adopted (yet) as Python so finding solutions to problems, or examples of how things are done, was a bit more challenging.

![backend flow](../published/images/backend_flow.png)
The basic workflow goes like this:
1. Write my blog post in Obsidian.
2. Push the final post to the GitHub repo that is synced with my blog directory.
3. A Github Action executes a bash script that sends the Markdown posts to the Go API using Curl
```bash
name: Publish Blog Posts
on:
  push:
    branches:
      - main
    paths:
      - '*.md' 
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Send Markdown posts to API
        env:
          API_URL: https://idleworkshop.com/api/posts
          API_TOKEN: ${{ secrets.BLOG_API_TOKEN }}
        run: |
          for file in *.md; do
            if [ -f "$file" ]; then
              TITLE=$(basename "$file" .md)
              CONTENT=$(cat "$file" | jq -Rs .)
              echo "Processing file: $file"
              echo "Title: $TITLE"
              echo "Content length: $(wc -c < "$file") bytes"
              
              curl -X POST "$API_URL" \
                -H "Content-Type: application/json" \
                -H "Authorization: $API_TOKEN" \
                -d '{"file": "'"$TITLE"'", "markdown": '"$CONTENT"'}' \
                -v
            fi
          done ```
4. The Go API then parses out the metadata and the content of the post.
```Go
func ParseMarkdown(markdown string) (*models.ParsedBlogPost, error) {
  // split the markdown into metadata and content
  log.Println(markdown)
  parts := strings.SplitN(markdown, "---", 3)
  
  if len(parts) < 3 {
    return nil, errors.New("Invalid markdown format")
  }
  // parse the metadata
  var metadata models.BlogMetadata
  if err := yaml.Unmarshal([]byte(parts[1]), &metadata); err != nil {
    return nil, err
  }
  // parse the content (everything below the frontmatter)
  content := strings.TrimSpace(parts[2])
  contentParts := strings.SplitN(content, "<!-- summary -->", 2)

  summary := strings.TrimSpace(contentParts[0])
  mainContent := ""
  if len(contentParts) > 1 {
    mainContent = strings.TrimSpace(contentParts[1])
  }
  
  return &models.ParsedBlogPost{
    Metadata: metadata,
    Summary: summary,
    Content:  mainContent,
  }, nil
}```
5. Lastly, the Go API sends all parsed pieces to the MongoDB collection.
```Go
func CreatePost(c *gin.Context, collection *mongo.Collection, ctx context.Context) {
  var markdownData struct {
    Markdown string `json:"markdown"`
    File string `json:"file"`
  }
  if err := c.ShouldBindJSON(&markdownData); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"oopsie!": err.Error()})
    return
  }
  log.Println(markdownData.Markdown)

  parsedPost, err := utils.ParseMarkdown(markdownData.Markdown)

  if err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"oopsie!": err.Error()})
    return
  }
  var slug = markdownData.File

  filter := bson.M{"slug": slug}
  update := bson.M{
    "$set": bson.M{
      "slug": slug,
      "title": parsedPost.Metadata.Title,
      "tags": parsedPost.Metadata.Tags,
      "nutshell": parsedPost.Metadata.Nutshell,
      "readtime": parsedPost.Metadata.ReadTime,
      "topic": parsedPost.Metadata.Topic,
      "content": parsedPost.Content,
      "summary": parsedPost.Summary,
    },
    "$setOnInsert": bson.M{"date": primitive.NewDateTimeFromTime(time.Now())},
  }

  options := options.Update().SetUpsert(true)

  result, err := collection.UpdateOne(ctx, filter, update, options)
  if err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"oopsie!": "Error upserting post"})
    return
      }

  if result.UpsertedCount > 0 {
    c.JSON(http.StatusCreated, gin.H{"message": "Post created"})
      } else {
    c.JSON(http.StatusOK, gin.H{"message": "Post updated"})
      }

} ```

## The Frontend - HTML/CSS and Vanilla JavaScript

I could have done something more exciting for the frontend stack like React or something similar. I've built a couple of React projects in the past. It's remarkably easy to get a good looking single page application with React, but my knowledge had a lot of holes due to never learning core HTML/CSS and DOM manipulation so I elected to keep things basic for my website and blog. My goal is to learn and explore what is possible with the vanilla web development languages.

So anyway moving right along...

When viewing the main {Blog} section of my site I wanted to show the posts with a little preview. 

Each preview consists of:
1. A Heading for the Title
2. Some metadata for the date, reading time, and the topic
3. A short description of the post which I call the "nutshell"
4. One or two paragraphs of a summary of the post
5. Some more metadata for the tags
6. And finally the button that the user can click on: "Check it out"

To get these 6 elements sorted out, I have to back it up to the start where I write the post itself. 

Even though the metadata is in the middle of the post preview sandwich, that's where the post begins. At the start of each post, there is a section for metadata called "frontmatter". The frontmatter is housed between two sets of triple dashes. 
Here is how it looks:
``` Markdown
---
title: Blog Format Template
tags:
  - markdown
  - webdev
  - html
  - test
  - TODO
nutshell: This is where the quick and dirty explanation goes.
topic: Tech, Automotive Repair, Building something, etc...
readtime: 10 minutes
--- 
```

The next element that is important to the preview is the summary. I didn't want to encapsulate this whole part in the frontmatter, though I certainly could have. For my purposes, the summary felt more like part of the post itself; an introduction if you will. So to make parsing this section easy I add a delimiter to the end of it. 

Everything after the frontmatter looks like this:
```
Here is where a summary will live. This is just going to be one or two paragraphs that lead the reader into the article. This is also what will display in the preview of the blog post on the main blogs page.

There might be multiple paragraphs in the summary. At this point, i'm not 100% sure how I want to differentiate the summary from the rest of the article. The challenge is that I need a reliable queue to parse to in the backend code. For now, I suppose i'll just use a series of characters as a line to separate the summary from the rest of the post.

<!-- summary -->
# Main Body.

Here is where the main content of the article (blog post) will reside. This can be a continuation from the summary, or could take off tangentially. Be free.
```

The tricky bit of rendering the Markdown neatly on the page is handled by [md-block](https://md-block.verou.me/#recipes). Md-block handles almost all the heavy lifting of converting my markdown files into HTML. Also, I love that the author created md-block mostly for fun, but also to meet his own needs instead of using something off the shelf. I have big respect for doing things the hard way.

Whether for the summary or the actual blog post, I put them in their own div and added that to the corresponding class. For example, here is a segment of my JavaScript code from the displayPost function:
``` JavaScript
function displayPost(post) {
  const blogPost = document.createElement("div");
  blogPost.classList.add("blog-post");

  const postContainer = document.getElementById("posts-container");
  postContainer.innerHTML = "";
  
  // A lot of code to handle metadata
  
  const contentElement = document.createElement("md-block");
  contentElement.textContent = post.content;
  postContainer.appendChild(blogPost);
  ```

By appending the md-block formatted contentElement to the postContainer, i'm able to access individual elements within the post for CSS formatting. For example:
``` CSS
.blog-post {
  width: 60%;
  /* Properties removed for brevity */
  padding: 2rem;
}
/* Example accessing the main text content of the post */
.blog-post p {
  margin-bottom: 0.5rem;
  color: #D6CFB4;
}
/* Example for unordered (bulleted) lists */
.blog-post ul {
  list-style-type: square;
  font-size: 1.2rem;
  padding: .5rem;
  color: #D6CFB4;
}
/* Example for images in blog post */
.blog-post img {
  width: 75%;
  padding: 1rem;
}
```

The full CSS and JS code are too lengthy to share in this post, but you can take a look at the repository for yourself at my [GitHub](https://github.com/anton-mcintosh/idleworkshop).  

## Future Plans

I've only just started scratching the surface of learning web development the hard way, so I have a lot of formatting and refinement to do on my UI. For one thing, i'm sure that i'll need to do some fudging of my CSS code for the first few blog posts to make sure the content displays correctly. 

This page is not optimized for mobile viewing yet. That is a whole can of worms. I'm sure I could burn another week getting everything to look right on small screens. I plan to do this, but I want to spend my time getting content developed first before I tackle this project. 

As for the backend side of things, i'd like to add a method of tracking the popularity of my posts to allow readers to filter based on that later on. I'm not sure if I want to record clicks or if I want people to actually click a "like" button. 

## Wrap it Up Already!

I set out on this adventure because I wanted to motivate myself to write more and to build something that showcased my ability to learn. What I've ended up with was a crash course in web development, cloud hosting, and the cathartic experience of building something from scratch.

For now I achieved my shorter-term goal of getting a somewhat automated blogging system and gaining a better understanding of the bedrock technology involved in web development. I'm proud of what I've accomplished so far and excited to carry on with this project and refine all aspects of my website. 

Most importantly, this blog is a reflection of my growth as a developer. It's also a demonstration that the best way to learn is simply to dive in and build something. 

If you stuck with this to the end, I would like to sincerely thank you for following along. If you've tackled a similar project, I'd love to hear about it. Either email me at anton@idleworkshop.com or reach out on [GitHub](http://github.com/anton-mcintosh)! Here's to making 2025 a year of continued learning and growth!