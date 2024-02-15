### HCS-1 Standard: File Data Management with Hedera Consensus Service

The HCS-1 standard provides a systematic approach to encode, chunk, upload, retrieve, and reassemble file data for applications using Hedera Consensus Service (HCS). This process is agnostic of the implementation details, focusing on the JSON structure and the use of a registry for efficient data management.

#### Topic Creation
 HCS-1 topics must include a SHA-256 hash of the file being uploaded as the memo. If the contents of the file are updated, the memo must be updated with the new SHA-256 hash to reflect those changes. Viewers of HCS-1 data will read the memo to understand if the file data is valid by comparing it to the original hash.

#### Encoding and Chunking

Before uploading to HCS, file data must be formatted correctly by following these three steps

1. The entire file should be compressed using zstd, this ensures cost-savings on all ends of the protocol in a lossless format.
  - Plenty of implementations for zSTD exist, for example, using NodeJS you could use the MongoDB library: https://github.com/mongodb-js/zstd/tree/main
  - ``` const compressedFile = await compress(file, 10);  ```
  - We recommend using a compression level of 10, but any compression level should work.
2. After compressing, the resulting data (typically a buffer) should be converted into a base64 string
3. The final base64 string should chunked into segments no greater than 1024 bytes. Each chunk is encapsulated in a JSON object with two attributes:

- `o`: The order index indicating the chunk's sequence in the overall file.
- `c`: The chunk's content, a substring of the base64-encoded file. The first chunk aka, `o = 0` should include a data prefix for the mime type.

The format for the data prefix is
```data:[mimeType];base64```
For example:
```data:image/png;base64,```

The format for each chunk in a message is as follows:
```json
{
  "o": 0,
  "c": "base64-encoded-chunk"
}
```

#### Uploading to HCS

Each chunk is uploaded to a Hedera Consensus Service topic as an HCS message. Keep in mind that because of the `o` property in the JSON schema, the sequence number that the chunk is uploaded in does not matter. This enables uploading many chunks in parallel.

#### Registry Use

A registry is employed to track uploaded data and their corresponding topic IDs on HCS. When data is uploaded, a registry entry can be created with the following structure to get an inscription # assigned:

```json
{
  "p": "hcs-1",
  "op": "deploy",
  "t_id": "topicId",
  "to": "walletAccountId"
}
```

- `p`: Protocol number
- `op`: The operation type, here "deploy" indicates registering a new upload.
- `t_id`: The topic ID where the file chunks are uploaded.
- `to`: The wallet account ID associated with the upload.

This entry is submitted to a predefined registry topic on Hedera, facilitating easy discovery and reference of uploaded files.

#### Retrieving and Reassembling

To display the data, the application retrieves all chunk messages from the specified HCS topic. Chunks are sorted by their order index (`o`) and concatenated based on their content (`c`). The combined base64 string is then decoded back into binary data for display.

The complete steps are:
1. Fetch all messages from the Topic ID, and sort them by their index `o`
2. Concatenate the messages together, and decompress the file using a `zstd` decoding library.
3. Ensure the memo of the Topic ID matches the SHA-256 hash of the decompressed file data.
4. Convert the uncompressed base64 string into binary data

#### Conclusion

The HCS-1 standard outlines a robust protocol for managing large file data sets within the Hedera Consensus Service, leveraging JSON for data structuring and a registry for efficient topic management. This approach ensures data integrity, facilitates easy data retrieval, and supports scalable application development on Hedera.