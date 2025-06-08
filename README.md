# Script to upload & use workflows on genomes in the Galaxy server
An easy and automatized way to:

1. Create a new Galaxy history, 

2. Upload the genome to that history, and 

3. Run the workflow on that genome

To do this, you must have:

1. A genome file (FASTA), and

2. A workflow on Galaxy

# Sections

Step-by-step

[Uploadin a single genome and running it through your Galaxy workflow](https://github.com/mathsampaio/use.galaxy-genomes/blob/main/Single_genome)

Looping over all genomes in a folder and run them through your Galaxy workflow


# Uploadin a single genome and running it through your Galaxy workflow

```{r}
# Step 0: Load Libraries and Set Credentials
library(httr)
library(jsonlite)

upload_and_run_workflow <- function(
  file_path,
  workflow_name = "workflow_name",
  history_name = NULL,
  galaxy_url = "https://usegalaxy.eu",
  api_key 
) {
  headers <- add_headers("x-api-key" = api_key)
  
# STEP 1: Create a history
  if (is.null(history_name)) {
    history_name <- paste0("Auto_", basename(file_path), "_", format(Sys.time(), "%Y%m%d%H%M%S"))
  }
  res_hist <- POST(
    url = paste0(galaxy_url, "/api/histories"),
    headers,
    body = list(name = history_name),
    encode = "json"
  )
  stop_for_status(res_hist)
  history_id <- content(res_hist)$id
  message("✔ History created: ", history_name)

# STEP 2: Upload the file
  res_upload <- POST(
    url = paste0(galaxy_url, "/api/tools"),
    headers,
    body = list(
      tool_id = "upload1",
      history_id = history_id,
      `files_0|file_data` = upload_file(file_path),
      `files_0|type` = "upload_dataset",
      `dbkey` = "?"
    ),
    encode = "multipart"
  )
  stop_for_status(res_upload)
  dataset_id <- content(res_upload)$outputs[[1]]$id
  message("✔ File uploaded: ", basename(file_path))
  
# STEP 3: Find workflow by name
  res_wf <- GET(paste0(galaxy_url, "/api/workflows"), headers)
  stop_for_status(res_wf)
  workflows <- content(res_wf)
  wf_match <- Filter(function(wf) wf$name == workflow_name, workflows)
  if (length(wf_match) == 0) stop("❌ Workflow not found: ", workflow_name)
  workflow_id <- wf_match[[1]]$id
  message("✔ Workflow found: ", workflow_name)

# STEP 4: Run the workflow
  input_map <- list("0" = list(id = dataset_id, src = "hda"))

  res_run <- POST(
    url = paste0(galaxy_url, "/api/workflows/", workflow_id, "/invocations"),
    headers,
    body = list(
      history_id = history_id,
      inputs = toJSON(input_map, auto_unbox = TRUE)
    ),
    encode = "json"
  )
  stop_for_status(res_run)
  invocation_id <- content(res_run)$id
  message("✔ Workflow launched! Invocation ID: ", invocation_id)

# Return all useful IDs
  return(list(
    history_id = history_id,
    dataset_id = dataset_id,
    workflow_id = workflow_id,
    invocation_id = invocation_id
  ))
}

```

## How to Run It
Once you’ve pasted the code above, you can run the batch like this:

```{r}
upload_and_run_workflow(
  file_path = "/your/path/to/fasta_files",
  api_key = "YOUR_GALAXY_API_KEY"
)
```

# Looping over all genomes in a folder and run them through your Galaxy workflow

```{r}
library(httr)
library(jsonlite)

upload_and_run_workflow <- function(
  file_path,
  workflow_name = "workflow_name",
  history_name = NULL,
  galaxy_url = "https://usegalaxy.eu",
  api_key
) {
  headers <- add_headers("x-api-key" = api_key)
  
  # STEP 1: Create a history
  if (is.null(history_name)) {
    history_name <- paste0("Auto_", basename(file_path), "_", format(Sys.time(), "%Y%m%d%H%M%S"))
  }
  res_hist <- POST(
    url = paste0(galaxy_url, "/api/histories"),
    headers,
    body = list(name = history_name),
    encode = "json"
  )
  stop_for_status(res_hist)
  history_id <- content(res_hist)$id
  message("✔ History created: ", history_name)

  # STEP 2: Upload the file
  res_upload <- POST(
    url = paste0(galaxy_url, "/api/tools"),
    headers,
    body = list(
      tool_id = "upload1",
      history_id = history_id,
      `files_0|file_data` = upload_file(file_path),
      `files_0|type` = "upload_dataset",
      `dbkey` = "?"
    ),
    encode = "multipart"
  )
  stop_for_status(res_upload)
  dataset_id <- content(res_upload)$outputs[[1]]$id
  message("✔ File uploaded: ", basename(file_path))

  # STEP 3: Find workflow by name
  res_wf <- GET(paste0(galaxy_url, "/api/workflows"), headers)
  stop_for_status(res_wf)
  workflows <- content(res_wf)
  wf_match <- Filter(function(wf) wf$name == workflow_name, workflows)
  if (length(wf_match) == 0) stop("❌ Workflow not found: ", workflow_name)
  workflow_id <- wf_match[[1]]$id
  message("✔ Workflow found: ", workflow_name)

  # STEP 4: Run the workflow
  input_map <- list("0" = list(id = dataset_id, src = "hda"))

  res_run <- POST(
    url = paste0(galaxy_url, "/api/workflows/", workflow_id, "/invocations"),
    headers,
    body = list(
      history_id = history_id,
      inputs = toJSON(input_map, auto_unbox = TRUE)
    ),
    encode = "json"
  )
  stop_for_status(res_run)
  invocation_id <- content(res_run)$id
  message("✔ Workflow launched! Invocation ID: ", invocation_id)

  return(list(
    genome = basename(file_path),
    history_id = history_id,
    dataset_id = dataset_id,
    workflow_id = workflow_id,
    invocation_id = invocation_id
  ))
}

# 🔁 BATCH SCRIPT
batch_run <- function(
  folder_path,
  file_extension = ".fna",
  api_key,
  output_csv = "galaxy_batch_log.csv"
) {
  fasta_files <- list.files(folder_path, pattern = paste0("\\", file_extension, "$"), full.names = TRUE)
  results <- list()
  
  message("🔍 Found ", length(fasta_files), " genome files to process...\n")
  
  for (i in seq_along(fasta_files)) {
    file <- fasta_files[i]
    message("🚀 [", i, "/", length(fasta_files), "] Processing: ", basename(file))
    tryCatch({
      res <- upload_and_run_workflow(file_path = file, api_key = api_key)
      results[[i]] <- res
    }, error = function(e) {
      message("❌ Error processing ", basename(file), ": ", e$message)
      results[[i]] <- list(genome = basename(file), error = e$message)
    })
    Sys.sleep(1)  # be kind to the Galaxy server
  }
  
  # Save results
  df <- do.call(rbind, lapply(results, as.data.frame))
  write.csv(df, file = file.path(folder_path, output_csv), row.names = FALSE)
  message("\n✅ Batch complete. Results saved to: ", output_csv)
}

```


## How to Run It
Once you’ve pasted the code above, you can run the batch like this:

```{r}
batch_run(
  folder_path = "/your/path/to/fasta_files",
  api_key = "YOUR_GALAXY_API_KEY"
)
```

✔️ You’ll see progress in the R console.
✔️ A .csv log file will be saved in the same folder, tracking history & invocation IDs for each genome.

# Explaining the function "upload_and_run_workflow()"
Let's break down the function "upload_and_run_workflow()" step-by-step. Here, I'll explain what each part does

## Step 1: Setup Headers and History

This step creates HTTP headers to authenticate with Galaxy using your API key and calls the Galaxy API to create a new history, where all uploaded data and workflow results will go.

If you don't give a custom history name, it auto-generates one with the name of the genome file. 

**You can set a custom name like this:**

```{r}
# upload_and_run_workflow(file_path = "...", api_key = "...", history_name = "MyAnalysis1")
```


## Step 2: Upload Genome File

This step uploads your genome file (FASTA) into the Galaxy history.
You only need to change:

**file_path** — your local file path to the genome

Optionally, set **dbkey** to something like "fungi" if needed for downstream tools

## Step 3: Find Your Workflow by Name

This step looks through your existing workflows and finds the one matching the name.

You can change **workflow_name** if your Galaxy workflow has a different name.

## Step 4: Run the Workflow

This step tells Galaxy to: run the workflow using the uploaded genome as input 0; And start the workflow now with this input and save the results in the new history

Note: This assumes your workflow uses a single FASTA input and it’s the first input slot (index 0). If your workflow has multiple inputs, we can update this structure.

## How to Run It

At least, you need to change only two things when calling the function:

upload_and_run_workflow(
  file_path = "**/path/to/your_genome.fasta**",
  api_key = "**YOUR_GALAXY_API_KEY**"
)

