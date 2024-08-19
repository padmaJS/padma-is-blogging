---
layout: post
title:  "Optimizing Large-Scale Data Backfills"
background: '/img/timezone.png'
subtitle: "Sharing my journey on how I made backfiller for millions of records"
---

# Backfilling Data at Scale!

## Introduction
In a recent project, I encountered a challenge that many developers face: backfilling data at scale. The task was to add a new field, **timezone**, to our **MyApp.Address** schema and populate it for millions of existing records in production. This post details my journey through this process, the challenges I faced, and the optimizations I implemented.

## What is Backfilling?

Backfilling is the process of adding or updating historical data in a database. It's often necessary when introducing new features or making schema changes that affect existing records. In our case, we needed to add timezone information to millions of address records.

## First iteration

Initially, I underestimated the complexity of updating millions of records. Here's the first version of the backfill function:
```elixir
defmodule MyApp.Tasks.BackfillTimezone do
  import Ecto.Query
  alias MyApp.Addresses.Address
  alias MyApp.Repo

  @batch_size 1000 # 1000 seemed like a reasonable number for a batch

  def run() do
    Repo.transaction(
      fn ->
        stream_addresses()
        |> Stream.chunk_every(@batch_size) # breaks down the records received into list of 1000 records
        |> Stream.each(&process_batch/1)
        |> Stream.run()
      end,
      timeout: :infinity
    )
  end

  defp stream_addresses do
    Address
    |> where([a], is_nil(a.timezone))
    |> Repo.stream(max_rows: @batch_size) # streams 1000 record from the database level to avoid memory overload 
  end

  # rest of the code is responsible for updating the timezone
  defp process_batch(addresses) do
    addresses
    |> Enum.map(&update_timezone/1)
  end

  defp update_timezone(%Address{geo_location: %{coordinates: coordinates}} = address) when not is_nil(coordinates) do
    timezone =
      case TzWorld.timezone_at(coordinates) do
        {:ok, timezone} -> timezone
        _ -> "UTC"
      end

    Ecto.Changeset.change(address, %{timezone: timezone}) |> Repo.update()
  end

  defp update_timezone(address), do: Ecto.Changeset.change(address, %{timezone: "UTC"}) |> Repo.update()
end
```
Thanks to Claude AI I was able to get an idea of how I was going to execute my backfill function. Let me explain quickly the process behind it. Get **Address** records which does not have a **timezone** yet. Use **Repo.stream** with **max_rows** option to get a 1000 rows from the database level, and using **Stream** chunk it to a list of 1000 records then update each record to have its timezone set.

So this ended up taking me 16 minutes for updating 100K records in my development environment. So by some basic match, it will take a couple of hours to backfill the data. Also I noticed that if there were to occur any problem in the midst of backfilling, the use of **Repo.transaction** would rollback all the changes made and start from zero again. I had to optimize the backfiller.

## Second iteration

Now being concerned with the keeping database connection available to others as well and also being resilient from failures. I had to refactor the code. During this refactor I stumbled upon [Backfilling Timezone Recursively](https://hexdocs.pm/oban/recursive-jobs.html#use-case-backfilling-timezone-data). This website piqued my interest. Here I saw that timezone data was being backfilled recursively and commit to 1 batch did not affect other batch. This seemed perfect and I refactored my code aroung it. Here's how it looks.
```elixir
defmodule MyApp.Tasks.BackfillTimezone do
  import Ecto.Query
  alias MyApp.Addresses.Address
  alias MyApp.Repo
  require Logger # added for process monitoring

  def run() do
    case fetch_batch_addresses_without_timezone() do
      [] ->
        :ok

      addresses ->
        process_batch(addresses)
        Logger.info("Completed batch")
        run()
    end
  end

  defp fetch_batch_addresses_without_timezone do
    Logger.info("Fetching batch")

    from(a in Address,
      where: is_nil(a.timezone),
      limit: 1000 # getting 1000 records in each call
    )
    |> Repo.all()
  end

  defp process_batch(addresses) do
    Logger.info("Updating timezone for batch")

    addresses
    |> Enum.group_by(&get_timezone/1)
    |> Enum.each(
      fn {timezone, addrs} ->
        ids = Enum.map(addrs, & &1.id)

        Address
        |> where([a], a.id in ^ids)
        |> Repo.update_all(set: [timezone: timezone]) # used update_all to update the timezone in a batch reducing the database calls
      end)
  end

  defp get_timezone(%{geo_location: %{coordinates: coordinates}}) do
    case TzWorld.timezone_at(coordinates) do
      {:ok, timezone} -> timezone
      _ -> "UTC"
    end
  end

  defp get_timezone(_), do: "UTC"
end
```
### Key Changes and Benefits

1.  Removed **Stream** and **Repo.transaction** to avoid long-lived transactions. Hence now the connection pool will be accessible to other processes as well. Also each batch is processed independently. So error in one batch wont cause problem to other batches.
2.  Used **limit: 1000** in the Ecto query itself to fetch batches of 1000 records at a time. There's no possibility of memory overload now.
3.  Implemented **Repo.update_all** for bulk updates, reducing database calls.
4.  Added logging for better visibility into the process.

## Performance Comparison

Surprisingly, the time taken to complete the backfill didn't improve significantly in my specific case. However, the optimized version addressed other critical issues like database connection management and error resilience.

## Lessons Learned

1.  Always consider scale when designing data migration or backfill processes.
2.  Long-running transactions can be problematic in production environments.
3.  Batch processing with independent commits can provide resilience against failures.

## Conclusion

While the performance gains were minimal in this specific case, the optimization process yielded valuable improvements in terms of resource management and fault tolerance. When dealing with large-scale data operations, it's crucial to consider not just speed, but also the impact on system resources and the ability to recover from failures.

Remember, the best approach may vary depending on your specific use case and system architecture. Always measure and test in an environment as close to production as possible before implementing large-scale data operations.