# Scoutfile sender

<!-- badges: start -->
[![Lifecycle: maturing](https://img.shields.io/badge/lifecycle-maturing-blue.svg)](https://www.tidyverse.org/lifecycle/#maturing)
<!-- badges: end -->

A cross-platform app for monitoring a volleyball scout file and making it available to remote users. It is intended as an open mechanism for sharing live-scouted data files (so that they can be used by coaches or others, in online apps or similar, as the match progresses).

The file is shared locally as well as over the internet. Internet sharing means that it can be accessed anywhere in the world (but only if the scout's laptop has internet access). Local sharing means that scouts without internet access can still share their file with other users on the same local network.

## Security note

For internet sharing, this app uses https://getpantry.cloud/ as its data exchange platform, because it is free to use and has a simple API for access. Note, however, that anyone who knows your pantry ID can see any file that you upload. Also be aware that once a file has been uploaded, its live data link has your pantry ID embedded in it. We therefore do not recommend using this app for uploading sensitive files. A more appropriate mechanism for sensitive data might be added at a later date, if there is a demand for it.

Science Untangled users can share live-scouted stats without exposing their pantry ID &mdash; see "Live stats" below.

## How to use

### Installation

1. Download and install the Scoutfile sender app from the [GitHub releases page](https://github.com/scienceuntangled/file_sender/releases). Installers are available for Windows, Mac, and Linux. If you are not a Science Untangled user, you can choose the app version without the SU live app link (it will show the data links only).

   Note that on some platforms you will get a warning when installing about "untrusted software" because we have not digitally signed the executables. If you are so inclined, you can [build the executable yourself](?tab=readme-ov-file#building-from-source) to be sure that they have not been tampered with.

1. For internet sharing, you will need a pantry ID. Go to https://getpantry.cloud/ and look for the "Create a Pantry" button. It will give you a pantry ID &mdash; save this somewhere.

## Usage

For both internet and local sharing:

1. Start the Scoutfile sender app, and then click the `Select scout file` button and choose your scout file.

    NOTE: it is best to point the file sender at your "safety scout" file (DataVolley) or "live export" file (VolleyStation) &mdash; these files are automatically saved at the end of each rally. The file sender will detect the updated file each time it is saved and re-upload it. If you point the file sender at a dvw file in your regular DataVolley "Seasons" directory you will need to remember to manually save the file whenever you want the updated data to be shared

### Internet sharing

For internet sharing:

2) Enter your pantry ID into the associated text box

2) The `Use base64 encoding` box is ticked by default, and is probably safest to leave that way. You might not need this if your file does not use any non-ASCII text (i.e. no accented, Cyrillic, kanji, or similar non-ASCII characters). Base64 encoding can cope with such text, but creates a larger file that will be slightly slower to process. If you see an error saying "stream did not contain valid UTF-8" then base64 encoding must be used. (Base64 encoding only matters for internet sharing, not local.)

2) The "Internet status" icon will show a progress indicator each time the file is uploading, followed by a green tick if the upload was successful or a red cross if not.

### Local sharing

For local sharing, you will need to be connected to a local wifi network. The bench laptop (or whatever client will be reading the live-scouted file) needs to be on the same network. Once you have selected a scout file to share, the Scoutfile sender will start a local webserver through which it delivers the file to those other users. The "Local status" icon will show a green tick or red cross.

If you wish to share the file locally but not over the internet, simple clear the pantry ID box.

## Accessing the live data

### Internet sharing

Once you have entered your pantry ID and selected a scout file, you should see the "Internet data link" button below. The data link is the Pantry download link, which will be something like `https://getpantry.cloud/apiv1/pantry/PANTRY_ID/basket/FILE_NAME` (note that the `FILE_NAME` will be URL-encoded, which means that e.g. spaces will be replaced by "%20").

Use the button to copy the associated link to the clipboard.

### Local sharing

The "Local network data link" is the link to the file on the local network, and will be something like `http://192.168.1.24:7474/live.dvw`. The `192.168.1.24` here is the IP address of the machine that the file sender is running on. Other machines on that same network can access this local link.

Use the button to copy the associated link to the clipboard.

NOTE: the IP address shown here might be incorrect if the scout laptop is connected to multiple local networks. In that case, you will need to determine the correct IP address to use manually. The data link will always be `http://SCOUT.IP.ADDRESS:7474/live.dvw`.

## Live stats

Science Untangled users can also open the "Live app" link (if using internet sharing). Once the live app has opened, ensure that you are logged into your Science Untangled account and then look for the "Share this session with anyone" button. This allows you to share stats from your live-scouted file with other (non-SU) users &mdash; your coaching staff, perhaps. The app will show a QR code to allow easy opening in a phone or other device.

### Downloading in scripts

You can download data from the file sender in your own programs.

#### From the internet-shared copy

The file is stored in json format, optionally base64-encoded. To retrieve it in a script you need to make a GET request, then extract the data from the json packet and optionally base64-decode it. A helper function in R might look like:

```
library(base64enc)
library(curl)
library(jsonlite)
library(httr)

fetch_pantry_url <- function(url, max_size = 5e6, accept = c("text/plain", "application/json")) {
    h <- new_handle(maxfilesize = max_size) ## control the max file size we will accept
    handle_setheaders(h, Accept = accept) ## and the allowed response types
    ## for pantry, download to memory, optionally b64-decode, and write to file
    url <- URLencode(url)
    res <- curl_fetch_memory(url = url, handle = h)
    if (res$status_code == 200) {
        res <- fromJSON(rawToChar(res$content))
        if (!setequal(names(res), c("filename", "data", "last_modified"))) {
            stop("pantry data in unexpected format")
        }
        path <- tempfile() ## save to file in temporary directory
        dir.create(path)
        path <- file.path(path, basename(res$filename))
        ## is the data base64-encoded?
        if (grepl("^([A-Za-z0-9+/]{4})*([A-Za-z0-9+/]{3}=|[A-Za-z0-9+/]{2}==)?$", res$data)) {
            ## yes
            cat(rawToChar(base64decode(res$data)), file = path, sep = "\n")
        } else {
            cat(res$data, file = path, sep = "\n")
        }
        path
    } else {
        ermsg <- tryCatch(paste0(": ", http_status(res$status_code)), error = function(e) "")
        stop("download failed", ermsg)
    }
}

x <- fetch_pantry_url("https://getpantry.cloud/apiv1/pantry/PANTRY_ID/basket/FILE_NAME")
## will download to a temporary file and return the filename

```

#### From the locally-shared copy

When sharing locally, the file is served as-is, so the download process is much simpler. In R you can just do something like:

```
myfile <- tempfile(fileext = ".dvw") ## temporary file
download.file("http://192.168.1.24:7474/live.dvw", destfile = myfile)

```

## Deleting old files

Pantry allows a maximum of 100 different files to be stored at any one time. The same file uploaded multiple times (with changes) in the same session only counts as one file.

Files will automatically be deleted from your Pantry storage after 30 days, but if you need to clean up your storage you can do so via the [Pantry dashboard](https://getpantry.cloud/) (click the "Dashboard" button).

## Building from source

Most users will use one of our [pre-built executables](https://github.com/scienceuntangled/file_sender/releases). However, if you wish to build the executable yourself (perhaps because an executable has not been provided for your machine, or you wish to modify the app):

1. Install Rust and Tauri: https://tauri.app/v1/guides/getting-started/prerequisites

1. Install the Tauri command line interpreter: `cargo install tauri-cli`

1. Clone this repository, change to the `src-tauri` directory and run the `cargo tauri build` command

1. The executable should be in the `src-tauri/target/release` directory.
