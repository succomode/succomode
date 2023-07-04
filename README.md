import sys
import json
import unicodedata
import urllib.request
import shutil

try:
    import cfscrape
except ImportError:
    print("error - required module 'cfscrape' not installed.", \
        "Use the command 'pip3 install cfscrape' to install")

if len(sys.argv) == 2 and sys.argv[1] == "--help":
    print("Usage: python3 patreonScript.py [-v] email password")
    print("Extracts (Shibby) audio information from a Patreon user's post stream.\n")
    print("  -v, --verify    verifies if an account is valid")
    print("      --help      display this help and exit")
    print("\nError codes:")
    print("  1  for invalid arguments")
    print("  2  for invalid username or password")
    print("  3  if device requires email verification")
    print("  4  for request failed; too many server requests")
    print("  5  for unknown error related to user account")
    print("  6  if user account is not a valid patron of u/ShibbySays")
    sys.exit(0);

if len(sys.argv) < 3 or len(sys.argv) > 4:
    print("usage: python3 patreonScript.py [-v] email password")
    sys.exit(1)

verify = 0
if len(sys.argv) == 4 and (sys.argv[1] == "-v" or \
        sys.argv[1] == "--verify"):
    verify = 1

email = sys.argv[1+verify]
password = sys.argv[2+verify]

# Attempt to create a new session using the user's Patreon account
scraper = cfscrape.create_scraper()
resp = scraper.post("https://api.patreon.com/login", \
    json={"data":{"email":email, "password":password}})
currentUser = json.loads(resp.content.decode("utf-8"))

cookie = resp.headers["Set-Cookie"]

VERIFICATION_POST_ID = 28614620    # Arbitrary patron-only post to verify user

verificationPost = json.loads(scraper.get("https://api.patreon.com/posts/" + \
    str(VERIFICATION_POST_ID)).content.decode("utf-8"))

if verify == 1 and "errors" not in currentUser and \
    "errors" not in verificationPost and \
        verificationPost["data"]["attributes"] \
        ["current_user_can_view"] == True:
    print("login valid")
    sys.exit(0)

if "errors" in currentUser:
    if currentUser["errors"][0]["code"] == 100:
        print("error - invalid username or password")
        sys.exit(2)
    elif currentUser["errors"][0]["code"] == 111:
        print("error - this device must be verified by clicking link in email")
        sys.exit(3)
    elif currentUser["errors"][0]["code_name"] == "RequestThrottled":
        print("error - too many requests")
        sys.exit(4)
    else:
        print("error - unknown")
        sys.exit(5)
elif "errors" not in verificationPost and \
            verificationPost["data"]["attributes"] \
            ["current_user_can_view"] == False:
        print("error - account not valid patron of u/ShibbySays")
        sys.exit(6)

SHIBBY_CAMPAIGN_ID = 322138    # Shibby's unique Patreon campaign ID

# Creates a direct download link for an attachment
def getFileUrl(postId, attachmentId):
    return "https://patreon.com/file?h={}&i={}" \
        .format(postId, attachmentId)

# Creates a list of direct download links for all attachments in a post
def getLinks(postData):
    links = []
    linkData = post["relationships"]["attachments"]["data"]
    for link in linkData:
        links.append(getFileUrl(postData["id"], link["id"]))
    return links

# Checks if a post is an audio post created by Shibby
def isShibbyAudioPost(postData):
    if postData["relationships"]["campaign"]["data"]["id"] == str(SHIBBY_CAMPAIGN_ID):
        tags = postData["relationships"]["user_defined_tags"]["data"]
        if len(tags) < 2:
            return False
        for tag in tags:
            if tag["id"].lower() == "user_defined;live event" or \
                tag["id"].lower() == "user_defined;meta":
                return False
    return True

# Attempts to parse a post's content into usable tags
def parseTags(content):
    tags = []
    if (content != None):
        tag = ""
        inStr = False
        for i in range(len(content)):
            c = content[i]
            if c == ']':
                tags.append(tag)
                tag = ""
                inStr = False
            elif c == '[':
                inStr = True
            elif c != '[' and inStr:
                tag += c
    return tags

print("Successfully authenticated with Patreon.")

postsJson = []
postNames = []
posts = json.loads(scraper.get("https://patreon.com/api/stream") \
    .content.decode("utf-8"))
while ("next" in posts["links"]):
    for post in posts["data"]:
        if isShibbyAudioPost(post):
            name = post["attributes"]["title"].lstrip().rstrip()
            links = getLinks(post)
            content = unicodedata.normalize("NFKD", post["attributes"]["content"])
            tags = parseTags(content)
            date = post["attributes"]["created_at"].split("T")[0]
            if name not in postNames:
                postJson = {"name":name, "links":links, "description":content, "tags":tags, "isPatreonFile":True, "type":"patreon"}
                postsJson.append(postJson)
                postNames.append(name)
                print("Downloading '" + name + "'...")
                i = 0
                displayName = name
                for link in links:
                    if (i > 0):
                        displayName = name + i
                    resp = scraper.get(link)
                    with open(displayName, "wb") as out_file:
                        out_file.write(resp.content)
                    i += 1
    nextPosts = "https://" + posts["links"]["next"]
    posts = json.loads(scraper.get(nextPosts).content.decode("utf-8"))


# https://patreon.com/api/campaigns/322138/posts?page[count]=<NUM_POSTS>    (retrieves posts/threads from a creator's campaign)
# https://patreon.com/api/stream    (retrieves current_user's posts from subscribed campaigns)
# https://patreon.com/file?h=<POST_ID>&i=<ATTACHMENT_ID>    (downloads an attachment; ex: audio file)
