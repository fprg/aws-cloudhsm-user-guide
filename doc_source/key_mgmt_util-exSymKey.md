# exSymKey<a name="key_mgmt_util-exSymKey"></a>

The `exSymKey` command in the key\_mgmt\_util tool exports a plaintext copy of a symmetric key from the HSM and saves it in a file on disk\. To export an encrypted \(wrapped\) copy of a key, use wrapKey\.

During the export process, `exSymKey` uses an AES key that you select \(the *wrapping key*\) to *wrap* \(encrypt\) and then *unwrap* \(decrypt\) the key to be exported\. However, the result of the export operation is a plaintext \(*unwrapped*\) key on disk\.

The `exSymKey` operation copies the key to a file that you specify, but it does not remove the key from the HSM or change its attributes\. You can export the same key multiple times\.

`exSymKey` exports only symmetric keys\. To export public keys, use exPubKey\. To export private keys, use exportPrivateKey\.

Before you run any key\_mgmt\_util command, you must start key\_mgmt\_util and login to the HSM as a crypto user \(CU\)\. 

## Syntax<a name="exSymKey-syntax"></a>

```
exSymKey -h

exSymKey -k <key-to-export>
         -w <wrapping-key>          
         -out <key-file>
         [-m 4] 
         [-wk <unwrapping-key-file> ]
```

## Examples<a name="exSymKey-examples"></a>

These examples show how to use `exSymKey` to export symmetric keys from your HSMs\.

**Example : Export a 3DES symmetric key**  
This command exports a Triple DES \(3DES\) symmetric key \(key handle 7\)\. It uses an existing AES key \(key handle 6\) in the HSM as the wrapping key\. Then, it writes the plaintext of the 3DES key to the `3DES.key` file\.  
The output shows that key 7 \(the 3DES key\) was successfully wrapped and unwrapped, and then written to the 3DES\.key file\.  
Although the output says that a "Wrapped Symmetric Key" was written to the output file, the output file contains a plaintext \(unwrapped\) key\.

```
        Command: exSymKey -k 7 -w 6 -out 3DES.key

       Cfm3WrapKey returned: 0x00 : HSM Return: SUCCESS

        Cfm3UnWrapHostKey returned: 0x00 : HSM Return: SUCCESS


Wrapped Symmetric Key written to file "3DES.key"
```

**Example : Exporting with session\-only wrapping key**  
This example shows how to use a key that exists only in the session as the wrapping key\. Because the key to be exported is wrapped, immediately unwrapped, and delivered as plaintext, there is no need to retain the wrapping key\.  
This series of commands exports an AES key with key handle 8 from the HSM\. It uses an AES session key created especially for the purpose\.  
The first command uses genSymKey to create an 256\-bit AES key\. It uses the `-sess` parameter to create a key that exists only in the current session\.  
The output shows that the HSM creates key 262168\.  

```
        Command:  genSymKey -t 31 -s 32 -l AES-wrapping-key -sess

        Cfm3GenerateSymmetricKey returned: 0x00 : HSM Return: SUCCESS

        Symmetric Key Created.  Key Handle: 262168

        Cluster Error Status
        Node id 1 and err state 0x00000000 : HSM Return: SUCCESS
```
Next, we run a series of checks to verify that key 8, he key to be exported, is a symmetric key that is extractable, and that the wrapping key, key 262168, is an AES key that exists only in the session\. You can use `findKey`, but this example exports the attributes of both keys to a file, and then uses grep to find the relevant attribute values in the file\.  
These commands use `getAttribute` with an `-a` value of `512` \(all\) to get all attributes for keys 8 and 262168\. For information about the key attributes, see the [[ERROR] BAD/MISSING LINK TEXT](key-attribute-table.md)\.  

```
getAttribute -o 8 -a 512 -out attributes/attr_8
getAttribute -o 262168 -a 512 -out attributes/attr_262168
```
These commands use `grep` to verify the attributes of the key to be exported \(key 8\) and the session\-only wrapping key \(key 262168\)\.  

```
    // Verify that the key to be exported is a symmetric key.
    $  grep -A 1 "OBJ_ATTR_CLASS" attributes/attr_8    
    OBJ_ATTR_CLASS
    0x04
   
    // Verify that the key to be exported is extractable.
    $  grep -A 1 "OBJ_ATTR_KEY_TYPE" attributes/attr_8
    OBJ_ATTR_EXTRACTABLE
    0x00000001

    // Find verify that the wrapping key is an AES key
    $  grep -A 1 "OBJ_ATTR_KEY_TYPE" attributes/attr_262168    
    OBJ_ATTR_KEY_TYPE
    0x1f

    // Verify that the wrapping key is a session key
    $  grep -A 1 "OBJ_ATTR_TOKEN" attributes/attr_262168
    OBJ_ATTR_TOKEN
    0x00    
    
    // Verify that the AES session key can be used for wrapping
     $  grep -A 1 "OBJ_ATTR_WRAP" attributes/attr_262168    
    OBJ_ATTR_WRAP
    0x00000001
```
Finally, we use an `exSymKey` command to export key 8 using the session key \(262168\) as the wrapping key\.  
When the session ends, key 262168 no longer exists\.  

```
        Command:  exSymKey -k 8 -w 262168 -out aes256_H8.key

        Cfm3WrapKey returned: 0x00 : HSM Return: SUCCESS

        Cfm3UnWrapHostKey returned: 0x00 : HSM Return: SUCCESS


Wrapped Symmetric Key written to file "aes256_H8.key"
```

**Example : Use an external unwrapping key**  
This example shows how to use an external unwrapping key to export a key from the HSM\.  
When you export a key from the HSM, you specify an AES key on the HSM to be the wrapping key\. By default, that wrapping key is used to wrap and unwrap the key to be exported\. However, you can use the `-wk` parameter to tell `exSymKey` to use an external key in a file on disk for unwrapping\. When you do, the key specified by the `-w` parameter wraps the target key and the key in the file specified by the `-wk` parameter unwraps the key\.   
Because the wrapping key must be an AES key, which is symmetric, the wrapping key in the HSM and unwrapping key on disk must be the same key\. To do this, you need to import the wrapping key to the HSM or export the wrapping key from the HSM before the export operation\.   
In this example, we will create a key outside of the HSM, import it to the HSM, and then use it to wrap and unwrap a symmetric key that is being exported\.  
The first command uses OpenSSL to generate a 256\-bit AES key\. It saves the key to the `aes256-forImport.key` file\. The OpenSSL command does not return any output, but you can use several commands to confirm its success\. This example uses the `wc` \(wordcount\) tool, which confirms that the file that contains 32 bytes of data\.  

```
$  openssl rand -out keys/aes256-forImport.key 32

$ wc keys/aes256-forImport.key
 0  2 32 keys/aes256-forImport.key
```
This command uses the `imSymKey` command to import the AES key from the `aes256-forImport.key` file to the HSM\. When the command completes, the key exists in the HSM with key handle `262167` and in the `aes256-forImport.key` file\.  

```
Command:  imSymKey -f keys/aes256-forImport.key -t 31 -l aes256-imported -w 6

        Cfm3WrapHostKey returned: 0x00 : HSM Return: SUCCESS

        Cfm3CreateUnwrapTemplate returned: 0x00 : HSM Return: SUCCESS

        Cfm3UnWrapKey returned: 0x00 : HSM Return: SUCCESS

        Symmetric Key Unwrapped.  Key Handle: 262167

        Cluster Error Status
        Node id 1 and err state 0x00000000 : HSM Return: SUCCESS
        Node id 0 and err state 0x00000000 : HSM Return: SUCCESS
```
This command uses the key in an export operation\.   
The command uses `exSymKey` to export key 21, a 192\-bit AES key\. To wrap the key, it uses key 212167 in the HSM\. To unwrap the key, it uses the same key in the `aes256-forImport.key` file\. When the command completes, key 21 is exported to the `aes192_h21.key` file\.  

```
        Command:  exSymKey -k 21 -w 262167 -out aes192_H21.key -wk aes256-forImport.key

        Cfm3WrapKey returned: 0x00 : HSM Return: SUCCESS

Wrapped Symmetric Key written to file "aes192_H21.key"
```

## Parameters<a name="exSymKey-params"></a>

**\-h**  
Displays help for the command\.   
Required: Yes

**\-k**  
Specifies of key handle of the symmetric key to export\. This parameter is required\. To find key handles, use the findKey command\.  
To verify that a key can be exported, use the getAttribute command to get the value of the `OBJ_ATTR_EXTRACTABLE` attribute, which is represented by constant `354`\.  
Required: Yes

**\-w**  
Specifies the key handle of the wrapping key\. This parameter is required\. To find key handles, use the findKey command\.  
A *wrapping key* is a key in the HSM that is used to encrypt \("wrap"\) and then decrypt \("unwrap\) the key to be exported\. Only AES keys can be used as wrapping keys\.  
You can use any AES key \(of any size\) as a wrapping key\. Because the wrapping key wraps, and then immediately unwraps, the target key, you can use as session\-only AES key as a wrapping key\. To determine whether a key can be used as a wrapping key, use getAttribute to get the value of the `OBJ_ATTR_WRAP` attribute, which is represented by the constant 262\. To create a wrapping key, use genSymKey to create an AES key \(type 31\)\.  
Required: Yes

**\-out**  
Specifies the path and name of the output file\. When the command succeeds, this file contains the exported key in plaintext\. If the file already exists, the command overwrites it without warning\.  
Required: Yes

**\-m**  
Specifies the wrapping mechanism\. The only valid value is **4**, which represents the `NIST_AES_WRAP` mechanism\.  
Required: No  
Default: 4

**\-wk**  
Use an external key to unwrap the key that is being exported\. Enter a file that contains a plaintext AES key\. The `-w` and `-wk` parameter values must resolve to the same plaintext key\.  
Required: No  
Default: Use the wrapping key on the HSM to unwrap\.

## Related Topics<a name="exSymKey-seealso"></a>

+ genSymKey

+ wrapKey