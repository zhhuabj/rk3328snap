```
mkdir model-assertion && cd model-assertion
sudo bash -c 'cat >rk3328-model.json' <<EOF
{
    "type": "model",
    "authority-id": "A0GXGvspAxfskf2wYVh1zf39UJbxNlGA",
    "brand-id": "A0GXGvspAxfskf2wYVh1zf39UJbxNlGA",
    "series": "16",
    "base": "core18",
    "model": "rock64",
    "display-name":"rk3328 snap",
    "architecture": "arm64",
    "gadget": "rk3328gadget",
    "kernel": "rk3328kernel",
    "timestamp": "2020-03-08T14:45:25+00:00"
}
EOF
snapcraft login
snap keys
#snap create-key my-key
snapcraft register-key my-key
cat rk3328-model.json |snap sign -k my-key > rk3328.model

#image building
sudo snap install --beta --devmode ubuntu-image
sudo ubuntu-image --channel edge \
--snap ../rk3328gadget/rk3328gadget_0.1_arm64.snap \
--snap ../rk3328kernelsnap/rk3328kernel_0.1_arm64.snap \
-O rk3328kernel.img ./rk3328.model
```

