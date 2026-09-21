HUMANE SAMPLER

Functional Specification

The product enables a music producer to transform a monophonic MIDI melody into a realistic audio performance played by real‑instrument samples. It achieves this by accepting two kinds of input: a bank of single‑note recordings taken from real instruments, and a melody expressed in MIDI. The system then selects, aligns and, when necessary, pitch‑shifts the recordings so that each MIDI note is rendered with the closest possible real‑world sample, finally exporting a single audio file together with a machine‑readable map of which recording landed at which moment in time.

Sample ingestion

The producer starts by uploading a single WAV recording—perhaps a scale, an improvisation or any uninterrupted performance—played on the instrument they wish to capture. Instead of requiring separate files for every pitch, the system analyses this long recording in five steps. First, it detects the onsets of individual notes. Secondly, it slices the audio at those onsets so that each note becomes its own fragment. Thirdly, it trims leading and trailing silence inside each fragment, ensuring the stored region contains only the useful sustain. Fourthly, it estimates the fundamental frequency of the fragment to determine its pitch. Finally, it stores the fragment on disk, tags it with the detected pitch and length and assigns a compact identifier. All fragments derived from the same source file are grouped into a collection representing that instrument. The producer can revisit any collection, inspect a grid that shows which of the thirty‑six target notes are covered, audition each fragment in the browser and, if desired, upload additional recordings to fill in missing notes.

Melody input

When the producer is ready to create a performance, they upload a monophonic MIDI file and choose one of the previously prepared instrument collections. The system reads every note event from the file, converting musical timing into absolute milliseconds so that each note is described with its pitch, start time and duration. For convenience the system also records the pitch of the previous and the next note, because that context later helps it choose the most suitable sample.


Matching strategy

For every note coming from the MIDI file the system looks for candidate recordings in the chosen collection. A recording is considered a candidate if its pitch is identical or at most one semitone away and its trimmed length is at least as long as the target note. Each candidate receives a compatibility score that rewards three factors in equal measure: similarity of length, whether the preceding note matches, and whether the following note matches. The recording with the highest score wins. If the winning recording is up or down by a single semitone, the system marks that difference so that it can be compensated later by a gentle pitch‑shift.

Audio assembly

Once every note has been paired with a recording, the system creates a timeline: each chosen recording must be placed at the note’s start time. Where a pitch‑shift is needed, a simple resampling step changes the playback speed so that the note sounds in tune. All snippets are concatenated, taking care that their internal silence offsets are honoured and that adjacent notes butt up without obvious clicks. At the very end a loudness normalisation pass keeps the overall level consistent.

Outputs delivered to the producer

The system delivers two artefacts. The first is a single WAV file that reproduces the complete melody as if played on the chosen instrument. The second is a structured file—JSON is a convenient format—that lists, for every note in the MIDI file, which recording was used, when it starts in the final audio and whether it was pitch‑shifted. A ZIP archive bundles the two so that the producer can download them with one click.

Interactive refinements

After hearing the automatically rendered result the producer may wish to refine the performance. In the browser they can click any note shown on a simple piano‑roll view. Doing so reveals the other candidate recordings ranked by their compatibility score. Choosing a different variant updates the mapping and triggers a partial re‑render so that only the affected note—or notes—are regenerated, keeping turnaround fast.

Operational considerations
The entire process must complete within roughly ten seconds for a one‑minute melody on modest server hardware. Uploaded recordings should be limited to a sensible size per file so that unintentional multi‑minute uploads do not overwhelm storage or processing time. To protect the service a fixed number of render jobs may run in parallel. Temporary render folders can be deleted after a fortnight so that storage does not grow without bound.

Stack for MVP

Backend: Python with FastAPI — ideal for the API and orchestration.


Audio Processing: Librosa & Pydub — best‑in‑class for the required audio tasks.


MIDI Parsing: Mido — simple and effective for MIDI input.


Frontend: React (with Next.js) — perfect for the interactive UI.


Database: PostgreSQL — necessary for structured metadata.


Task Queue: Celery with Redis — crucial for non‑blocking background processing.


File Storage: Local server filesystem (MVP only) — simplifies setup by removing cloud storage; all files are read from and written to directories on the same server where the application runs.



Workflow

   1. Sample Ingestion:


       The producer uploads a WAV file via the React frontend.


       The FastAPI backend receives the file and saves it to a designated directory on the server, for example: /srv/humane_sampler/uploads/<collection_id>/source.wav.


       FastAPI creates a job in the Celery queue. The job data will contain the local file path to the new WAV file.


       A Celery worker picks up the job. It reads the file directly from the local path.


       Librosa and Pydub are used to process the audio. The resulting note fragments are saved into another local directory, e.g., /srv/humane_sampler/fragments/<collection_id>/<sample_id>.wav.


       The PostgreSQL database stores the local path to each fragment alongside its pitch and length metadata.


   2.Audio Assembly & Delivery:


       When a render is requested, the Celery worker is given a list of local file paths for the required samples (retrieved from PostgreSQL).


       It reads these files from the disk, assembles the new performance, and generates the final WAV and JSON files.


       These output files are saved to a temporary local directory, e.g., /srv/humane_sampler/renders/<render_id>/.


       The final ZIP archive is created from this directory and saved locally.


       The FastAPI backend serves the final ZIP file directly to the user for download.


Key Considerations

While this simplifies the MVP, it's important to be aware of the trade-offs and how to plan for the future.

1. Plan for Future Migration
   
The biggest challenge with local storage is moving to a multi-server or cloud-native environment later. To make this transition easy, you should abstract your file operations.

Recommendation: Create a simple StorageManager class in your Python code.
      For the MVP, this class will have methods like save() and get_path() that simply read and write to the local filesystem using Python's built-in os or pathlib modules.


      Your application code (FastAPI and Celery) will only call your StorageManager, not the filesystem functions directly.


      When you're ready to scale, you can write a new S3StorageManager class with the exact same methods, but inside they will use a cloud SDK (like boto3 for AWS S3). You can then switch which    manager is used with a single configuration change, without rewriting your application logic.


2. Ensure Data Persistence
   
If you deploy your MVP on a cloud server (like an AWS EC2 instance), be sure to attach a persistent block storage volume (like EBS) and store your files there. Do not use the server's "instance store" or "ephemeral disk", as all your data will be lost if the server is stopped or restarted.

4. Scalability is Limited
5. 
This architecture will only work as long as your web server (FastAPI) and your background processor (Celery) run on the same machine. If you need to scale them independently onto different servers, they will no longer share a common filesystem, and the process will break. This is the primary reason to eventually move to a shared object storage solution like S3.
This local storage approach is a great, pragmatic choice for getting the humane_sampler MVP built and tested quickly.
