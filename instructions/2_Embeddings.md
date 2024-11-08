# Embeddings

This session will explore one of the basic tools in AI app development, *embeddings*.

An embedding model converts some input - usually text or an image - into a numerical vector (e.g., an array of floats). This vector represents the *meaning* of the input, not the characters in the source string or the pixel colors in an image. Nearby vectors have similar meanings.

## Get your embedding model ready

For this exercise we'll use Ollama, so make sure you have it installed. Embedding models can be small, and can run quickly even on CPU on any laptop.

Pull the the [all-minilm](https://ollama.com/library/all-minilm) embedding model, which is pretty small and general-purpose:

```
ollama pull all-minilm
```

... and leave Ollama running:

```
ollama serve
```

## Open the project

Open the project `exercises/Embeddings/Begin`.

In `Program.cs`, notice that there are three different entry points:

```cs
await new SentenceSimilarity().RunAsync();
//await new ManualSemanticSearch().RunAsync();
//await new FaissSemanticSearch().RunAsync();
```

Leave `SentenceSimilarity` uncommented, because that's where we'll start.

Open `SentenceSimilarity.cs`. First check you can generate an embedding for some text. Add this at the bottom of the `RunAsync` method:

```cs
var embedding = await embeddingGenerator.GenerateAsync("Hello, world!");
Console.WriteLine($"Embedding dimensions: {embedding[0].Vector.Span.Length}");
foreach (var value in embedding[0].Vector.Span)
{
    Console.Write("{0:0.00}, ", value);
}
Console.WriteLine();
```

If you run this, you should see it produces a vector of length 384. It's a *normalized* vector (i.e., the sum of its squares adds to 1) so it represents a direction in 384-dimensional space. Any sentence that has similar meaning will be in a similar direction, and vice-versa.

## Compute similarity

To compute the similarity between two embeddings, many different metrics are possible.

The most commonly used is *cosine similarity*, which gives a number from -1 to 1 (higher means more similar). It's simply the cosine of the angle between the two vectors. Remember that cos(0)=1, so if the angle between two vectors is zero, this formula will return 1.

Compute similarity over a few strings as follows:

```cs
var catVector = (await embeddingGenerator.GenerateAsync("cat"))[0].Vector;
var dogVector = (await embeddingGenerator.GenerateAsync("dog"))[0].Vector;
var kittenVector = (await embeddingGenerator.GenerateAsync("kitten"))[0].Vector;

Console.WriteLine($"Cat-dog similarity: {TensorPrimitives.CosineSimilarity(catVector.Span, dogVector.Span):F2}");
Console.WriteLine($"Cat-kitten similarity: {TensorPrimitives.CosineSimilarity(catVector.Span, kittenVector.Span):F2}");
Console.WriteLine($"Dog-kitten similarity: {TensorPrimitives.CosineSimilarity(dogVector.Span, kittenVector.Span):F2}");
```

You should see that "cat" is more related to "kitten" than it is to "dog".

## Semantic search

You can use this technique to find the closest text to a given search term. Let's do this for a set of documents for employees at a company, dealing with HR policies and such like. This company only has ~100 such policies, so we can easily implement semantic search in memory without needing any vector database or advanced indexing.

Back in `Program.cs`, comment out the line for `SentenceSimilarity` and uncomment the one for `ManualSemanticSearch`.

Now, in `ManualSemanticSearch.cs`, at the bottom of `RunAsync`, add:

```cs
var embeddingsResult = await embeddingGenerator.GenerateAsync(TestData.DocumentTitles.Values);
```

`TestData.DocumentTitles` is the dictionary of HR document titles, and `embeddingsResult` is a list of their embeddings. Let's bring these together into a single data structure holding titles and corresponding embeddings:

```cs
// Can also be done with .Zip, but is less obvious
var docInfoWithEmbeddings = TestData.DocumentTitles.Select((docTitle, index) => new
{
    Id = docTitle.Key,
    Text = docTitle.Value,
    Embedding = embeddingsResult[index].Vector,
}).ToList();
Console.WriteLine($"Got {docInfoWithEmbeddings.Count} title-embedding pairs");
```


If you want to see what you just did, set a breakpoint on the `Console.WriteLine` line and inspect `docInfoWithEmbeddings`.

Next let's implement a search REPL:

```cs
while (true)
{
    Console.Write("\nQuery: ");
    var input = Console.ReadLine()!;
    if (input == "") break;

    // TODO: Compute embedding and search
}
```

Replace the `TODO` comment with the following. First compute the embedding of the current `input`:

```cs
var inputEmbedding = (await embeddingGenerator.GenerateAsync(input))[0];
```

And now loop over all the `docInfoWithEmbeddings` candidates. For each one, compute the similarity to the search term, and  order by similarity descending:

```cs
var closest =
    from candidate in docInfoWithEmbeddings
    let similarity = TensorPrimitives.CosineSimilarity(
        candidate.Embedding.Span, inputEmbedding.Vector.Span)
    orderby similarity descending
    select new { candidate.Text, Similarity = similarity };
```

Yes, it's LINQ query syntax! You don't see it a lot these days but it looks nice for something like this. If you want to re-express that with a load of `.Select` and lambdas, go for it. (But what a waste of time.)

Finally, display the closest three:

```cs
foreach (var result in closest.Take(3))
{
    Console.WriteLine($"({result.Similarity:F2}): {result.Text}");
}
```

Try any HR-like search term, such as:

 * exercise
 * where can I park
 * how to get my boss fired

Also try spelling mistakes. And opposites.

## Optional: Implement the similarity calculation manually

Cosine similarity is a really simple algorithm. And given that you're working with normalized vectors, it's mathematically identical to a dot product. This simply means multiplying together the corresponding elements in the two vectors, and adding together the results.

To prove this to yourself, try implementing it:

```cs
private static float DotProduct(ReadOnlySpan<float> a, ReadOnlySpan<float> b)
{
    // TODO: Implement this
}
```

<details>
  <summary>SOLUTION</summary>

  ```cs
  private static float DotProduct(ReadOnlySpan<float> a, ReadOnlySpan<float> b)
  {
      var result = 0f;
      for (int i = 0; i < a.Length; i++)
      {
          result += a[i] * b[i];
      }
      return result;
  }
  ```
</details>

Verify you can use this instead of `TensorPrimitives.CosineSimilarity` and still get the same result.

If you're really keen, have a go at vectorizing it (e.g., using `Vector256.Multiply`). There's a solution in `exercises/Embeddings/End`. Is it faster than your unvectorized version? How does the speed compare with `TensorPrimitives.CosineSimilarity`?

## Optional: Semantic opposites

If embedding similarity is measured from -1 to +1, what are the most semantically different two strings you can find? Can you find any string pair whose similarity is close to -1? What about two opposite-meaning statements?