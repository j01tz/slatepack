# slatepack
This is a test repo to experiment with a standard for generating armored slates for Grin transactions.

It should be able to:

- [x]  Serialize armored strings from `grin-wallet` prepared slate json strings and file bytes

- [x]  Deserialize armor prepared by `slatepack` as bytes or json string for use in `grin-wallet`

### Sending Armored Grin (From Binary Serialized Slates)
1. Create a new rust project with `cargo new grin-armor` and `cd grin-armor`
2. In `Cargo.toml` under `[Dependencies]` add `slatepack = { git = "https://github.com/j01tz/slatepack" }`
3. In `/src/main.rs` add the following lines:
```
use slatepack;
...
let mut file = File::open("src/test.tx.bin").unwrap();
let mut buffer = Vec::new();
file.read_to_end(&mut buffer).unwrap();
let mut slate = slatepack::SerializedSlate::new();
slate.is_bin = true;
slate.bin = buffer;
let armor = slatepack::SerializedSlate::armor(&slate).unwrap();
println!("{}", armor.data);
```
4. Create a legacy slate file: `grin-wallet --floonet send -d "/path/to/grin-armor/test.tx" -m binfile 1.337`
5. Next execute `cargo run` and the armor string will be printed to your terminal window
6. Share the armored string with your counterparty and have them read the next section
7. When they have returned the response string you can save a `test.armor.response` file and access with code like above or directly as a pasted string:
```
use std::fs::File;
use std::io::Write;
...
let armor_response = "BEGINSLATEPACK...ENDSLATEPACK.".to_string();
let armor_format = slatepack::ArmoredSlate::new(&armor_response, true);
let response_slate = slatepack::ArmoredSlate::remove_armor(armor_format).unwrap();
let mut response_file = File::create("test.tx.response").unwrap();
response_file.write_all(response_slate.data.as_bytes()).unwrap();
```
8. Now finalize and broadcast the tx file with `grin-wallet --floonet finalize -i test.tx.response`

### Receiving Armored Grin (From Binary Serialized Slates)
1. Start a new rust project with slatepack dependency as in steps in previous section
2. Use some code like below to handle the armored string you receive:
```
use slatepack;
use std::fs::File;
use std::io::Write;
...
let armor = "BEGINSLATEPACK...ENDSLATEPACK.".to_string();
let armored_slate = slatepack::ArmoredSlate::new(&armor, true);
let slate = slatepack::ArmoredSlate::remove_armor(&armored_slate).unwrap();
let mut slate_file = File::create("test.tx").unwrap();
slate_file.write_all(slate.data.as_bytes()).unwrap();
```
3. Next execute `cargo run` and the transaction file will be saved as `test.tx` in `grin-armor/`.
4. Import the transaction file into your wallet: `grin-wallet --floonet receive -i /path/to/grin-armor/test.tx`.
5. `grin-wallet` will create a `test.tx.response` file that you need to armor to return to your counterparty.
6. Modify the above code to read and armor the response file to give you a copy pastable string
```
let mut file = File::open("src/test.tx.bin").unwrap();
let mut buffer = Vec::new();
file.read_to_end(&mut buffer).unwrap();
let mut slate = slatepack::SerializedSlate::new();
slate.is_bin = true;
slate.bin = buffer;
let armor = slatepack::SerializedSlate::armor(&slate).unwrap();
println!("{}", armor.data);
```
7. You can now copy and paste the armored slate returned in your terminal to your counterparty

### How slatepack armoring works when serializing with `SerializedSlate::armor()`
1. Raw slate data provided by `grin-wallet` must be built as a `SerializedSlate` to allow distinguishing json and bin serialization
  - Convert json string to bytes if necessary and indicate whether the started slate is serialized as binary with `is_bin`
2. An error checking code is generated by taking the first four bytes of a double sha256 hash of the slate binary
3. Error checking code bytes and slate bytes are concatenated
4. Output from step 3 is base58 encoded
5. Output from step 4 is formatted into 15 character words for readability
6. Output from step 5 is framed with a header and footer containing human and machine readable characters, notably a period at the end of the header, a period at the start of the footer and a period at the end of the footer.

### How slatepack armoring works when deserializing with `ArmoredSlate::remove_armor()`
1. Armor data string is deserialized into bytes
2. Bytes are consumed until the first byte representing a "period" is reached- this is the framing header
3. Framing header is validated
4. Remaining bytes after the framing header are consumed until the next byte representing a "period" is reached- this is the slate body
5. Any remaining bytes until the next "period" byte are the framing footer
6. Framing footer is validated
7. Slate body bytes are sanitized for whitespace characters
8. Slate body bytes are decoded with base58
9. First four bytes of the result are the error check code
10. Remaining slate body bytes are twice hashed with SHA256 and the first four bytes are compared with the error check code from the previous step for validation
11. If the validation is successful the slate body bytes should match those from the original slate before it was armored
12. Returned value can then be written to a file when using or made ready for transport as a json string when using `remove_armor()` for use in `grin-wallet`

#### V4 initial slate json string:
```
{\n  \"ver\": \"4.3\",\n  \"sta\": \"S1\",\n  \"id\": \"mavTAjNm4NVztDwh4gdSrQ\",\n  \"amt\": \"1000000000\",\n  \"fee\": \"8000000\",\n  \"sigs\": [\n    {\n      \"excess\": \"02ef37e5552a112f829e292bb031b484aaba53836cd6415aacbdb8d3d2b0c4bb9a\",\n      \"nonce\": \"03c78da42179b80bd123f5a39f5f04e2a918679da09cad5558ebbe20953a82883e\"\n    }\n  ]\n}\n
```

#### V4 initial slate armored with slatepack:
```
BEGINSLATEPACK. 2xfX9bS82gxJ6jN D6X4X843wrT84DT FkstYawTtacDqeU HybZLwcF26YXCix bmpTcw3hii6BF4x axfussSBrZq7xMQ P1rbw3GpebXkMeY i7aSjRZgxqDJwzt MyGqBauGHxEFZNg FeEbVFsqXaKkKwK PQdxrpKutVmJV67 pbY4nbeZgPtRaZj QZL61Wj7iGqKBuu tDvEwUBsuhb9GRf 1MK3jegnbKG5JJr QVrYignWoZrpXUx PiDobVMLh7RTRrz T6GNKJftiwJ5gup f7T69mFG9H8JqCG A4i5ogfcHhfgg5b 2AzBJA49nh39Pyh zotpGBj7a7RK4Kr bWqksP7iTxvfdUB zVwinrRjLeryvF7 uroTKm514ZDDrKf ZbyncaZXcFGYHWM tWp5ccsjjtM1JqB adragavjHQyjqkU 2JH9YnoRkx2AyuU qvn7nnb4fMTAVSw sbAPwBTua7njNht nhzRqtdHTr9KM9q eXHDN9iascu. ENDSLATEPACK.
```

### TODO
- [ ] Add support for multiple slatepack messages for very big slates
- [ ] Make more idiomatic

This should not be used for any real transactions. It is for educational purposes to visualize and experiment with the armored slate string format.
