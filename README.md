# letsencryptaws Cookbook

This cookbook is for an implementation of SSL certificate generation and fetching via the Let's Encrypt certificate authority. Certificates are synced from local storage to S3, which is then used by nodes to retrieve the generated certificate. Authentication is done via DNS challenges and the offical Certbot plugin for Route 53.

Nodes do not need to be EC2 instances to retrieve or request certificates. All that is required is AWS credentials or profile to perform Route 53 and S3 operations.

## Requirements

- Python 3 or 2.7 (for certbot and awscli)
- certbot ACME client
- Domain(s) hosted by AWS Route 53
- S3 bucket for storing/retrieving certificate files
- Properly stored AWS credentials (see https://github.com/aws/aws-sdk-ruby#configuration)

### Platforms

Certificate generation:
- Ubuntu

The goal for certificate retrieval is to support Windows but for now, Ubuntu only.

### Cookbooks

- `remote_file_s3` - To grab certificates from S3
- `pyenv` - For grabbing Python environment that certbot recipe uses to install `awscli` and `certbot`.

## Usage

### letsencryptaws::default

Set the `certs` attribute as described below and then include this recipe in your cookbook or run_list.

**NOTE** You should manually generate a default certificate (self-signed/fake CA) and place the key, certificate and CA certificate at the `"/#{node['letsencryptaws']['sync_path']}/default-ssl"` path in your S3 sync bucket. These will act as stand-ins for the real certificates/key until they are generated by `letsencryptaws::certbot` after they are first "requested" by a node. So in theory, your real certificates will take up to (chef interval + splay * 3) until they land on the requesting node.

The flow looks like this:
- First Chef run on requesting node. Attribute (`['letsencryptaws']['certs']['example.com'] = []`) gets saved to server (probably created dynamically by a cookbook). Default cert/key gets saved to node from S3.
- Chef runs on host with 'letsencryptaws::certbot' in its run_list. Requests certificate and uploads to S3.
- Second Chef run on requesting node overwrites the previously saved default cert/key with real cert/key from S3.

Any service that uses a certificate provided by this recipe should subscribe to one of the certificate file resources so that it can be reloaded when the certificate is renewed. For example:

```
service 'nginx' do
  action %i[start enable]
  subscribes :restart, "file[#{::File.join(node['letsencryptaws']['ssl_cert_dir'], 'example.com.crt')}]", :delayed
end
```

**FOR WILDCARD CERTIFICATES**, you should specify the CN as you'd like as `certs` key (e.g. `*.example.com`) however any `*` will be substituted with `star` in the filenames to prevent the need for escaping. So the filename you'd reference for the certificate would be `star.example.com.crt`.

### letsencryptaws::certbot

This is meant to be run by a single host that manages fetching certificates based on a Chef server `search`. Make sure the instance profile or AWS access keys in the data bag is granted the following permissions on the domains in which you allow certificates to be requested by nodes:

- `route53:ChangeResourceRecordSets`
- `route53:ListHostedZonesByName`
- `route53:GetChange`

The credentials will also require write access to the S3 bucket and path that you choose to sync to.

If you desire persistent storage on an EBS volume, use the `['letsencryptaws']['ebs_device']` to specify the path to the device. This will device will have an ext4 filesystem created on it if one does not already exist and be mounted at `['letsencryptaws']['config_dir']`. This is where certbot will store its configs and certificates. All operations take place locally at this path and at the end of the recipe gets synced to S3.

Certbot operations use the `--expand` and `--cert-name` arguments to keep the certificates up-to-date with the requested names. This means the certificate will be renewed appropriately as nodes desire for the certificate name changes.

### letsencryptaws::import_keystore

This recipe takes certificates and imports them into a Java keystore.

## Attributes

### letsencryptaws::default

For certificate retrieval, just specify what certificates you would like by common name
of the certificate and an array of Subject Alternative Names for the cert. The certificate
may have additional SANs if other nodes request them for the same common name.

<table>
  <tr>
    <th>Key</th>
    <th>Type</th>
    <th>Description</th>
    <th>Default</th>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['certs']</tt></td>
    <td>Hash</td>
    <td>keys are the common name, values are an array of strings that are the SANs for the cert. These all get merged together in the final certificate.</td>
    <td><tt>{}</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['ssl_cert_dir']</tt></td>
    <td>string</td>
    <td>path where ssl certs will be downloaded to</td>
    <td><tt>/etc/ssl/certs</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['ssl_key_dir']</tt></td>
    <td>string</td>
    <td>path where ssl private keys will be downloaded to</td>
    <td><tt>/etc/ssl/private</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['ssl_ca_dir']</tt></td>
    <td>string</td>
    <td>path where ssl CA certificates will be downloaded to</td>
    <td><tt>/etc/ssl/certs</tt></td>
  </tr>
</table>

### letsencryptaws::import_keystore

This recipe is automatically included if the `import_keystore` hash is not empty.

<table>
  <tr>
    <th>Key</th>
    <th>Type</th>
    <th>Description</th>
    <th>Default</th>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['import_keystore']</tt></td>
    <td>Hash</td>
    <td>keys are full paths to Java keystores, values are an array of primary names of certificates to add to the keystore</td>
    <td><tt>{}</tt></td>
  </tr>
</table>

### letsencryptaws::certbot

<table>
  <tr>
    <th>Key</th>
    <th>Type</th>
    <th>Description</th>
    <th>Default</th>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['config_dir']</tt></td>
    <td>string</td>
    <td>dir where all certbot configuration will be stored, including certs</td>
    <td><tt>/mnt/letsencrypt</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['certbot_version']</tt></td>
    <td>string</td>
    <td>version to enforce for certbot</td>
    <td><tt>1.18.0</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['certbot_dns_version']</tt></td>
    <td>string</td>
    <td>version to enforce for certbot-dns-route53</td>
    <td><tt>['letsencryptaws']['certbot_version']</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['data_bag']</tt></td>
    <td>string</td>
    <td>Name of data bag used for credentials storage</td>
    <td><tt>nil</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['data_bag_item']</tt></td>
    <td>string</td>
    <td>Name of item within data bag for credentials storage</td>
    <td><tt>nil</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['email']</tt></td>
    <td>string</td>
    <td>Email addressed used for certbot during generation</td>
    <td><tt>nobody@example.com</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['ebs_device']</tt></td>
    <td>string</td>
    <td>device of the ebs volume to mount on `config_dir` (only applies on ec2 instances)</td>
    <td><tt>/dev/xvdf</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['test_certs']</tt></td>
    <td>boolean</td>
    <td>request certs from staging (signed by fake CA, subject to less rate limiting)</td>
    <td><tt>false</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['remove_unused_certs']</tt></td>
    <td>boolean</td>
    <td>remove certificates that are no longer requested by any node</td>
    <td><tt>true</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['sync_bucket']</tt></td>
    <td>string</td>
    <td>s3 bucket to sync local certificate directory to</td>
    <td><tt>nil</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['sync_path']</tt></td>
    <td>string</td>
    <td>path on the `sync_bucket` to sync certificate directory to</td>
    <td><tt>letsencrypt</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['kms_key_id']</tt></td>
    <td>string</td>
    <td>UUID of the kms key to use for server-side encryption (optional)</td>
    <td><tt>nil</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['blocklist']</tt></td>
    <td>array of strings</td>
    <td>Exact matches of primary certificate name to prevent generation</td>
    <td><tt>[]</tt></td>
  </tr>
  <tr>
    <td><tt>['letsencryptaws']['link_pybins']</tt></td>
    <td>boolean</td>
    <td>Link <tt>aws</tt> and <tt>certbot</tt> in /usr/local/bin to pyenv paths if true
    <td><tt>true</tt></td>
  </tr>
</table>

## Data Bags

A data bag is used to store sensitive credential information for AWS and Java keystores. You can arbitrarily specify the name and item name with `node['letsencryptaws']['data_bag']` and `node['letsencryptaws']['data_bag_item']` attributes. If you do not wish to use a data bag, you can place the credentials following the same hash structure at `node.run_state['letsencryptaws_creds']`.

The keys inside the data bag item can be:

<table>
  <tr>
    <th>Key</th>
    <th>Type</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><tt>aws_access_key_id</tt></td>
    <td>string</td>
    <td>AWS_ACCESS_KEY_ID for storing/fetching certificates from S3</td>
  </tr>
  <tr>
    <td><tt>aws_secret_access_key</tt></td>
    <td>string</td>
    <td>AWS_SECRET_ACCESS_KEY for storing/fetching certificates from S3</td>
  </tr>
  <tr>
    <td><tt>keystore_passwords</tt></td>
    <td>hash</td>
    <td>Keys are paths to Java keystore files, values are the passwords to them. One special key is `default` which will be used as a catch-all password if a keystore does not have a specific entry.</td>
  </tr>
  <tr>
    <td><tt>p12_password</tt></td>
    <td>string</td>
    <td>Password to use when generating pkcs12 keyring files.</td>
  </tr>
</table>

## License and Authors

Authors: Matt Kulka <matt@lqx.net>
