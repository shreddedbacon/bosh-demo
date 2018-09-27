Run `bucc up` to create a local director using virtualbox and BOSH lite

This can take some time to run while it downloads resources, starts a virtual machine in virtualbox and configures it as a director.
```bash
bucc up
```

Once complete, you can get the rest of the tools if you need them (for this we aren't using them)
```bash
bucc uaa
bucc credhub
```

Then run bosh env to display the director status
```bash
bosh env
```
Will produce the following output
```
Using environment '192.168.50.6' as client 'admin'

Name      bucc  
UUID      fc8ae04d-165d-467c-9895-271dad173ec5  
Version   268.0.1 (00000000)  
CPI       warden_cpi  
Features  compiled_package_cache: disabled  
          config_server: enabled  
          dns: disabled  
          snapshots: disabled  
User      admin
```

And set up the host routes so your computer knows how to access the vms inside virtualbox
```bash
bucc routes
```

!!! note
    A lot of the next steps for this introduction will be run from within the bucc directory for easy clean up at the end.
