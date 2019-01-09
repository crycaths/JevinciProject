# FootPrintMap

- 소속 : Jevinci
- 개발인원 : 2명(서버개발자, 클라이언트개발자)
- 개발기간 : 1년
- 개발 플렛폼 : iOS(iPhone)
- 개발 환경 : Swift3.0, MangoDB, SpringBoot, AWS
- 앱 개발 내용 : 여유 시간과 목적지로 지역의 명소를 추천하여 도보 여행 네비게이션을 제공합니다.
- 현재상태 : 재정문제로 개발중단

## 구조
![image info](./serverstructure.png)

## 기능
- 경로 데이터는 T Map API를 사용하여, 지도는 GoogleMap으로 표현
- 도보여행 네비게이션
- 인상깊은 여행 경로 minimap으로 저장
- 여러 사용자가 실시간으로 위치 공유

# 기능의 코드리뷰

### NETWORK
##### Task.swift
<details><summary>CLICK ME</summary>
<p>

```swift
//
//  NonAuth.swift
//  fpm
//
//  Created by Je.vinci.Inc on 2017. 10. 12..
//  Copyright © 2017년 Crycat. All rights reserved.
//

import Foundation
import Alamofire
import SwiftyJSON

protocol Multipart{
}
protocol Imageable {
}
protocol Task: Multipart, Imageable{
}

extension Task{
    private func convert(request: Request) -> URLRequest{
        let urlstr = request.path
        let encoded = urlstr.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed)
        let url = URL(string: encoded!)
        let header = request.headers

        let parameter = request.parameters
        let method = request.method
        var urlrequest = URLRequest(url: url!)
        urlrequest.httpMethod = method.rawValue
        if header != nil {
            for (key, value) in header!{
                urlrequest.addValue(value as! String, forHTTPHeaderField: key)
            }
        }

        if parameter != nil{
            let data = try? JSONSerialization.data(withJSONObject: parameter!, options: [])
            urlrequest.httpBody = data
        }
        return urlrequest
    }
    func excute(request:Request,completion: @escaping (JSON?) -> ()){
        Alamofire.request(convert(request: request)).responseJSON{ response in
            switch response.result {
            case .success(let value):
                let json = JSON(value)
                NSLog("success: \(json)")
                completion(json)
            case .failure(let error):
                NSLog("fail: \(error)")
                completion(nil)
            }
        }
    }
}

extension Multipart{
    func excuteMulti(request: Request, filename: String, filedata: Data, mimetype: MimeType, completion: @escaping (JSON?) -> ()){
        guard let request = self.createMultipartURLRequest(request: request, filename: filename, filedata: filedata, mimetype: mimetype) else {return completion(nil)}
        Alamofire.request(request).responseJSON{ response in
            switch response.result {
            case .success(let value):
                let json = JSON(value)
                NSLog("success: \(json)")
                completion(json)
            case .failure(let error):
                NSLog("fail: \(error)")
                completion(nil)
            }
        }

    }
    private func createMultipartURLRequest(request: Request, filename: String, filedata: Data, mimetype: MimeType) -> URLRequest?{
        guard let url = URL(string: request.path) else {return nil}
        var result: URLRequest = URLRequest(url: url)
        let bound = self.generateBoundaryString()
        let oldheader = request.headers
        let header = ["Content-type": "multipart/form-data; boundary=\(bound)",
            "accept": "application/json"]
        let method = request.method
        guard let parameter = request.parameters else {return nil}
        let body: Data = self.createMultipartBody(parameter: parameter, filename: filename, filedata: filedata, mimetype: mimetype, boundary: bound)
        for (key, value) in header{
            result.addValue(value, forHTTPHeaderField: key)
        }
        if oldheader != nil{
            for (key, value) in oldheader!{
                let stringvalue = value as? String
                result.addValue(stringvalue ?? "", forHTTPHeaderField: key)
            }
        }
        result.httpMethod = method.rawValue
        result.httpBody = body
        return result
    }
    private func generateBoundaryString() -> String{
        return "Boundary-\(NSUUID().uuidString)"
    }
    private func createMultipartBody(parameter: Dictionary<String,Any>, filename: String, filedata: Data, mimetype: MimeType, boundary: String) -> Data{
        let body = NSMutableData()
        let name = "file"
        for (key,value) in parameter{
            body.appendString(string: "--\(boundary)\r\n")
            body.appendString(string: "Content-Disposition: form-data; name=\"\(key)\"\r\n")
            body.appendString(string: "Content-Type: application/json; charset=UTF-8\r\n\r\n")
            let stringvalue = value as? String
            body.appendString(string: "\(stringvalue ?? "")\r\n")
        }
        body.appendString(string: "--\(boundary)\r\n")
        var mime: String
        switch mimetype {
        case .png:
            mime = "image/png"
        case .jpeg:
            mime = "image/jpeg"
        }
        let defFileName = filename
        body.appendString(string: "Content-Disposition: form-data; name=\"\(name)\"; filename=\"\(defFileName)\"\r\n")
        body.appendString(string: "Content-Type: \(mime)\r\n\r\n")
        body.append(filedata)
        body.appendString(string: "\r\n")
        body.appendString(string: "--\(boundary)--")
        return body as Data
    }
}

extension Imageable{
    private func convertRequestToURLRequest(request: Request) -> URLRequest?{
        guard let url = URL(string: request.path) else {return nil}
        var urlrequest: URLRequest = URLRequest(url: url)
        if request.headers != nil {
            for (key, value) in request.headers!{
                urlrequest.addValue(value as! String, forHTTPHeaderField: key)
            }
        }
        urlrequest.httpMethod = "GET"
        return urlrequest
    }
    func excuteImage(request: Request,completion: @escaping (Data?) -> ()){
        guard let urlrequest = convertRequestToURLRequest(request: request) else {return}
        URLSession.shared.dataTask(with: urlrequest){ data, response, error in
//            guard let httpresponse = response as? HTTPURLResponse else {return}
//            guard httpresponse.statusCode == 200 else {NSLog("code : \(httpresponse.statusCode)");return completion(nil)}
            guard let imagedata = data else {return completion(nil)}
            completion(imagedata)
            }.resume()
    }
}
enum MimeType{
    case png
    case jpeg
}
extension NSMutableData {
    func appendString(string: String) {
        let data = string.data(using: String.Encoding.utf8, allowLossyConversion: true)
        append(data!)
    }
}
struct CrycatTask : Task{
}
```

</p>
</details>

##### Service.swift
<details><summary>CLICK ME</summary>
<p>

```swift
//
//  Service.swift
//  fpm
//
//  Created by Je.vinci.Inc on 2017. 10. 12..
//  Copyright © 2017년 Crycat. All rights reserved.
//

import Foundation
//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE
//let url_jevinci = "http://localhost:8080"
//let url_jevinci = "http://192.168.0.4:8080"
//let url_jevinci = "http://localhost:8080"
//let url_jevinci = "http://localhost:8080"
let url_jevinci = "http://server.jevinci.io"
//let url_jevinci = "http://172.16.3.204:8080"
//let url_jevinci = "http://192.168.0.31:8080"
//let url_jevinci = "http://192.168.0.31:8080"
let jevinci_storage = "/files/User"
let SK_base = "https://apis.skplanetx.com/tmap"
let SK_etc = "version=1&reqCoordType=WGS84GEO&resCoordType=WGS84GEO"
let SK_etc_geo = "version=1&coordType=WGS84GEO"
let SK_search = "/pois?"
let SK_detail = "/pois"
let SK_r_geocoding = "/geo/reversegeocoding?"
let SK_path = "/routes/pedestrian?"
let SK_appKey = "&appKey=dcca17d6-b5fc-32b5-991f-5240083ddf16"
//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE//URL RESOURCE

enum ServRequest: Request{
    case deletePartyUser(partyUserId: String)
    case addfriend(master: User,friend: User)
    case searchUser(nick: String)
    case searchUserWithId(id: String)
    case friendlist(jtoken: JevinciTokenModel,master: User)
    case login(apitoken: APITokenModel)
    case nickcheck(nickname: String)
    case refreshtoken(jevincitoken: JevinciTokenModel)
    case signup(signupmodel: SignUpModel)
    case profile(jevincitoken: JevinciTokenModel)
    case image(path: String)
    case searchplaces(placename: String)
    case reverseGeocodint(lat: String, lon: String)
    case direction(startplace: Place,endplace: Place)
    case recommand(jtoken: JevinciTokenModel, resource: FPMResource)
    case saveroute(token: JevinciTokenModel, fpmdata: FPMData)
    case deletefavorite(favoriteid: String)
    case savefavorite(token: JevinciTokenModel, fpmdata: FPMData)
    case loadFavorites(userid: String)
    case loadMinionWithUserId(userid: String, name: String)
    case loadThumbnail(path: String)
    case loadFPMRouteWithId(routeId: String)
    case createParty(party: FPMParty)
    case saveDevice(device: Device)
    case loadInvitationWithFPMPushId(fpmpush: FPMPush)
    case acceptparty(invitation: FPMInviation)
    case currentPartyUser(partyid: String)
    case reverseGeocording(place: Place)

    var path: String{
        switch self {
        case .reverseGeocording(let place):
            return SK_base + SK_r_geocoding + SK_etc_geo + "&lat=\(place.latitude)&lon=\(place.longitude)" + SK_appKey + "&addressType=A10"
        case .deletePartyUser(let id):
            return url_jevinci + "/party/user/\(id)"
        case .currentPartyUser(let partyid):
            return url_jevinci + "/party/user/update/\(partyid)"
        case .acceptparty:
            return url_jevinci + "/party/invite/answer"
        case .deletefavorite(let id):
            return url_jevinci + "/favorites/delete/\(id)"
        case .loadFavorites(let id):
            return url_jevinci + "/favorites/findByUserId/\(id)"
        case .login:
            return url_jevinci + "/login"
        case .refreshtoken:
            return url_jevinci + "/auth/jwt/refresh"
        case .nickcheck(let nick):
            return url_jevinci + "/common/isAvailableNickname/\(nick)"
        case .signup:
            return url_jevinci + "/common/signup"
        case .profile:
            return url_jevinci + "/user/profile"
        case .image(let path):
            return path
        case .searchplaces(let placename):
            return SK_base + SK_search + SK_etc + "&searchKeyword=\(placename)" + SK_appKey
        case .reverseGeocodint(let lat,let lon):
            return SK_base + SK_r_geocoding + SK_etc_geo + "&lat=\(lat)&lon=\(lon)" + SK_appKey + "&addressType=A10"
        case .direction(let start,let end):
            return SK_base + SK_path + SK_etc + SK_appKey + "&startX=\(start.longitude)" + "&startY=\(start.latitude)" + "&endX=\(end.longitude)" + "&endY=\(end.latitude)" + "&startName=\(start.title)" + "&endName=\(end.title)"
        case .recommand:
            return url_jevinci + "/common/footDraw"
        case .friendlist( _,let user):
            return url_jevinci + "/friend/findByUserId/\(user.id)"
        case .searchUser(let nick):
            return url_jevinci + "/user/findByKeyword/\(nick)"
        case .addfriend:
            return url_jevinci + "/friend/add"
        case .saveroute:
            return url_jevinci + "/route/save"
        case .savefavorite:
            return url_jevinci + "/favorites/add"
        case .loadMinionWithUserId(let userid,let name):
            return url_jevinci + jevinci_storage + "/\(userid)/route/\(name)"
        case .loadThumbnail(let path):
            return path
        case .loadFPMRouteWithId(let routeId):
            return url_jevinci + "/route/findOne/\(routeId)"
        case .createParty:
            return url_jevinci + "/party/create"
        case .saveDevice:
            return url_jevinci + "/common/device/save"
        case .loadInvitationWithFPMPushId(let push):
            return url_jevinci + "/party/invitation/\(push.invitation_id)"
        case .searchUserWithId(let id):
            return url_jevinci + "/user/find/\(id)"
        }
    }
    var headers: [String : Any]?{
        switch self {
        case .login,.nickcheck, .signup, .saveDevice:
            return ["Content-type":"application/json"]
        case .refreshtoken(let token):
            return ["refreshToken":"\(token.refreshToken)","Content-type":"appication/json"]
        case .profile(let jtoken), .friendlist(let jtoken, _):
            return ["Content-type":"application/json", "jwt-header":"\(jtoken.accessToken)"]
        case .recommand(let data):
            return ["Content-type":"application/json", "jwt-header":"\(data.jtoken.accessToken)"]
        case .searchUser, .addfriend, .savefavorite, .createParty, .searchUserWithId, .loadFavorites, .deletefavorite, .acceptparty, .currentPartyUser, .deletePartyUser:
            //after edit about token lazy?
            let token = JevinciTokenModel.getToken()
            let accesstoken = token?.accessToken ?? ""
            return ["Content-type":"application/json", "jwt-header":"\(accesstoken)"]
        case .loadMinionWithUserId, .loadThumbnail, .loadFPMRouteWithId, .loadInvitationWithFPMPushId:
            let token = JevinciTokenModel.getToken()
            let act = token?.accessToken ?? ""
            return ["jwt-header":"\(act)"]
        case .saveroute(let token, _):
            return ["jwt-header":"\(token.accessToken)"]
        default:
            return nil
        }
    }
    var parameters: [String : Any]?{
        switch self {
        case .login(let token):
            return token.toDic()
        case .signup(let Signup):
            return Signup.toDic()
        case .recommand(let data):
            return data.resource.toJSON()
        case .addfriend(let master,let friend):
            var dic: Dictionary<String,Any> = [:]
            dic["userId"] = master.jevinciPost()["id"]
            dic["friend"] = friend.jevinciPost()
            return dic
        case .saveroute(_,let fpmdata):
            return fpmdata.jevinciPost()
        case .savefavorite(_ ,let fpmdata):
            return fpmdata.favoritePost()
        case .createParty(let party):
            return party.toJSON()
        case .saveDevice(let device):
            return device.jevinciPost()
        case .acceptparty(let invitation):
            return invitation.jevinciPost()
        default:
            return nil
        }
    }
    var method: HTTPMethod{
        switch self {
        case .login,.signup, .recommand, .addfriend, .saveroute, .savefavorite, .createParty, .saveDevice, .acceptparty:
            return .post
        case .deletefavorite, .deletePartyUser:
            return .delete
        default:
            return .get
        }
    }
}

class Service{
    static var instance = Service()

    private init(){}
    private var task = CrycatTask()

    private func requestLogin(apitoken: APITokenModel, completion: @escaping (Int) -> ()){
        task.excute(request: ServRequest.login(apitoken: apitoken)){ tempjson in
            guard let json = tempjson else {return completion(1001)}
            if let statuscode = json["status"].int {
                switch statuscode{
                case 101:
                    //네이버토큰에러
                    completion(101)
                case 401:
                    //유저 X
                    completion(401)
                default:
                    //예외
                    completion(1002)
                }
            }else{
                //유저 O
                if let json = tempjson {
                    if let accessToken = json["accessToken"].string, let refreshToken = json["refreshToken"].string{
                        let jtoken = JevinciTokenModel(act: accessToken, rft: refreshToken, date: Date())
                        jtoken.setJevinciToken()
                        completion(501)
                    }else{
                        completion(1002)
                    }
                }else{
                    completion(1002)
                }
            }
        }
    }
    private func requestLoginDoubleForJevinciToken(apitoken: APITokenModel, completion: @escaping (JevinciTokenModel?) -> ()){
        task.excute(request: ServRequest.login(apitoken: apitoken)){ tempjson in
            guard let json = tempjson else {return completion(nil)}
            let jevincitoken = JevinciTokenModel(act: json["accessToken"].stringValue, rft: json["refreshToken"].stringValue, date: Date())
            completion(jevincitoken)
        }
    }
    private func requestAccesstokenWithRefreshtoken(token: JevinciTokenModel, completion: @escaping (JevinciTokenModel?) -> ()){
        task.excute(request: ServRequest.refreshtoken(jevincitoken: token)){ tempjson in
            guard let json = tempjson else {return completion(nil)}
            guard let act = json["accessToken"].string else {return completion(nil)}
            let new = JevinciTokenModel(act: act, rft: token.refreshToken, date: Date())
            completion(new)
        }
    }
    func searchPlacesBySKapi(placename: String, completion: @escaping (Array<Place>?) -> ()){
        task.excute(request: ServRequest.searchplaces(placename: placename)){ tempjson in
            guard let json = tempjson else {return}
            guard let searchPoiInfo = json["searchPoiInfo"].dictionary else {return}
            guard let pois = searchPoiInfo["pois"]?.dictionary else {return}
            guard let poi = pois["poi"]?.array else {return}
            let places = poi.map{ json in
                return Place(SKObject: json)
            }
            completion(places)
        }
    }
    func login(apitoken: APITokenModel, completion: @escaping (Int) -> ()){
        requestLogin(apitoken: apitoken){ code in
            completion(code)
        }
    }
    func twiceloginforJevinciToken(apitoken: APITokenModel, completion: @escaping (JevinciTokenModel?) ->()){
        requestLoginDoubleForJevinciToken(apitoken: apitoken){ temp in
            completion(temp)
        }
    }
    func refreshToken(jevinciToken: JevinciTokenModel, completion: @escaping (JevinciTokenModel?) -> ()){
        requestAccesstokenWithRefreshtoken(token: jevinciToken){ tempjevincitoken in
            guard let newtoken = tempjevincitoken else {return completion(nil)}
            newtoken.setJevinciToken()
            completion(newtoken)
        }
    }
    func checkNickname(nickname: String, completion: @escaping (String?) -> ()){
        task.excute(request: ServRequest.nickcheck(nickname: nickname)){ temp in
            guard let json = temp else {return completion(nil)}
            guard let boolean = json.bool else {return completion(nil)}
            guard boolean == true else {return completion(nil)}
            completion(nickname)
        }
    }
    func signup(signUpModel: SignUpModel, completion: @escaping(User?) -> ()){
        task.excute(request: ServRequest.signup(signupmodel: signUpModel)){ temp in
            guard let json = temp else {return completion(nil)}
            let tempuser = User(jsonObject: json)
            completion(tempuser)
        }
    }
    func profile(jtoken: JevinciTokenModel, completion: @escaping(User?) -> ()){
        task.excute(request: ServRequest.profile(jevincitoken: jtoken)){ temp in
            guard let json = temp else {return completion(nil)}
            let user = User(jsonObject: json)
            user.setUser()
            completion(user)
        }
    }
    func reversGeocodingBySKapi(lat: String, lon: String, completion: @escaping (Place?) -> ()){
        task.excute(request: ServRequest.reverseGeocodint(lat: lat, lon: lon)){ temp in
            guard let json = temp else {return}
            guard let detail = json["addressInfo"].dictionary else {return}
            guard let fullAddress = detail["fullAddress"]?.string else {return}
            guard let dlat = Double(lat) else {return}
            guard let dlon = Double(lon) else {return}
            let place = Place(lat: dlat, lon: dlon, title: fullAddress, address: fullAddress)
            completion(place)
        }
    }
    func requestRecommandation(jtoken: JevinciTokenModel, resource: FPMResource, completion: @escaping (FPMData?) -> ()){
        task.excute(request: ServRequest.recommand(jtoken: jtoken, resource: resource)){ temp in
            guard let json = temp else {return}
            let fpmdata = FPMData(jsonObject: json)
            completion(fpmdata)
        }
    }
    func requestDirection(sp: Place, ep: Place, completion: @escaping (Array<Feature>?) -> ()){
        task.excute(request: ServRequest.direction(startplace: sp, endplace: ep)){ temp in
            guard let json = temp else {return}
            guard let features = json["features"].array else {return}
            var path: Array<Feature> = []
            for feature in features{
                guard let geometry = feature["geometry"].dictionary else {return}
                guard let checktype = geometry["type"]?.string else {return}
                switch checktype {
                case "Point":
                    let point = Point(jsonObject: feature)
                    path.append(point)
                case "LineString":
                    let line = LineString(jsonObject: feature)
                    path.append(line)
                default:
                    break
                }
            }
            completion(path)
        }
    }
    func requestDirections(fpmdata: FPMData, completion: @escaping (Array<Direction>?) -> ()){
        let totalcount = fpmdata.waypoints.count - 1
        guard totalcount != -1 else {return}
        var directions: Array<Direction> = []{
            didSet{
                if directions.count == totalcount{
                    directions.sort{$0.sequence < $1.sequence}
                    completion(directions)
                }
            }
        }
        for i in 1..<fpmdata.waypoints.count{
            let sp = fpmdata.waypoints[i - 1].place
            let ep = fpmdata.waypoints[i].place
            self.requestDirection(sp: sp, ep: ep){ temp in
                guard let path = temp else {return}
                let direction = Direction(sequence: i - 1, path: path)
                directions.append(direction)
            }
        }
    }
    func requestFriendList(master: User, completion: @escaping (Array<User>?) -> ()){
        JevinciTokenModel.getToken{ temp in
            guard let token = temp else {return}
            self.task.excute(request: ServRequest.friendlist(jtoken: token, master: master)){ temp in
                guard let json = temp else {return}
                guard let list = json.array else {return}
                let friends = list.map{ (json) -> User in
                    let user = json["friend"]
                    return User(jsonObject: user)
                }
                completion(friends)
            }
        }
    }
    func searchUserWithNick(nick: String, completion: @escaping (Array<User>?) -> ()){
        task.excute(request: ServRequest.searchUser(nick: nick)){ temp in
            guard let json = temp else {return}
            guard let jsonusers = json.array else {return}
            let users = jsonusers.map{ (json) -> User in
                return User(jsonObject: json)
            }
            completion(users)
        }
    }
    func addFriend(friend: User, completion: @escaping (Bool) -> ()){
        User.getUser{ temp in
            guard let user = temp else {return completion(false)}
            self.task.excute(request: ServRequest.addfriend(master: user, friend: friend)){ temp in
                guard temp != nil else {return completion(false)}
                completion(true)
            }
        }
    }
    //Multipart
    func saveRoute(miniondata: Data, minionname: String, completion: @escaping (FPMData?) -> ()){
        JevinciTokenModel.getToken{ temp in
            guard let token = temp else {return completion(nil)}
            guard let fpmdata = FPMManager.getInstance.fpmdata else {return completion(nil)}
            self.task.excuteMulti(request: ServRequest.saveroute(token: token, fpmdata: fpmdata), filename: minionname, filedata: miniondata, mimetype: MimeType.png){ temp in
                guard let json = temp else {return}
                let fpmdata = FPMData(jsonObject: json)
                completion(fpmdata)
            }
        }

    }
    func saveFavorite(fpmdata: FPMData, completion: @escaping (Int?) -> ()){
        JevinciTokenModel.getToken{ temp in
            guard let token = temp else {return completion(nil)}
            self.task.excute(request: ServRequest.savefavorite(token: token, fpmdata: fpmdata)){ tempjson in
                guard let json = tempjson else {return completion(nil)}
                let id = json["id"].intValue
                completion(id)
            }
        }

    }
    func loadMinion(imagename: String, completion: @escaping (Data?) -> ()){
        User.getUser{ tempuser in
            guard let user = tempuser else {return}
            self.task.excuteImage(request: ServRequest.loadMinionWithUserId(userid: String(user.id), name: imagename)){ tempdata in
                guard let data = tempdata else {return completion(nil)}
                completion(data)
            }
        }
    }
    func loadThumbnail(path: String, completion: @escaping (Data?) ->()){
        task.excuteImage(request: ServRequest.loadThumbnail(path: path)){ tempdata in
            guard let data = tempdata else {return completion(nil)}
            completion(data)
        }
    }
    func loadRoute(routeId: String, completion: @escaping (FPMData?) -> ()){
        task.excute(request: ServRequest.loadFPMRouteWithId(routeId: routeId)){ tempjson in
            guard let json = tempjson else {return completion(nil)}
            let route = FPMData(jsonObject: json)
            completion(route)
        }
    }
    func createParty(party: FPMParty, completion: @escaping (FPMParty?) -> ()){
        task.excute(request: ServRequest.createParty(party: party)){ temp in
            guard let json = temp else {return completion(nil)}
            let party = FPMParty(json: json)
            completion(party)
        }
    }
    func saveDevice(device: Device, completion: @escaping (Device?) -> ()){
        task.excute(request: ServRequest.saveDevice(device: device)){ temp in
            guard let json = temp else {return completion(nil)}
            let device = Device(jsonObject: json)
            completion(device)
        }
    }
    func loadInvitation(push: FPMPush, completion: @escaping (FPMInviation?) -> ()){
        task.excute(request: ServRequest.loadInvitationWithFPMPushId(fpmpush: push)){ temp in
            guard let json = temp else {return completion(nil)}
            let invitation = FPMInviation(json: json)
            completion(invitation)
        }
    }
    func loadUser(id: String, completion: @escaping (User?) -> ()){
        task.excute(request: ServRequest.searchUserWithId(id: id)){ temp in
            guard let userjson = temp else {return completion(nil)}
            let user = User(jsonObject: userjson)
            completion(user)
        }
    }
    func loadFavorites(userid: Int, completion: @escaping (Array<FPMData>?,Array<Int>?) -> ()){
        task.excute(request: ServRequest.loadFavorites(userid: String(userid))){ temp in
            guard let json = temp else {return completion(nil,nil)}
            guard let datas = json.array else {return completion(nil,nil)}
            var favoritesId: Array<Int> = []
            let fpmdatas = datas.map{ json -> FPMData in
                let route = json["route"]
                let id = json["id"].intValue
                favoritesId.append(id)
                return FPMData(jsonObject: route)
            }
            completion(fpmdatas,favoritesId)
        }
    }
    func deleteFavorite(favoriteid: String, completion: @escaping (Bool) -> ()){
        task.excute(request: ServRequest.deletefavorite(favoriteid: favoriteid)){ temp in
            guard let json = temp else {return completion(false)}
            completion(true)
        }
    }
    func acceptFPMParty(invitation: FPMInviation, completion: @escaping (User?) -> ()){
        task.excute(request: ServRequest.acceptparty(invitation: invitation)){ tempuser in
            guard let json = tempuser else {return completion(nil)}
            let jsonuser = json["user"]
            let user = User(jsonObject: jsonuser)
            guard user.nickname.count > 0 else {return completion(nil)}
            completion(user)
        }
    }
    func updatePartyUser(partyid: String, completion: @escaping ([User]?) -> ()){
        task.excute(request: ServRequest.currentPartyUser(partyid: partyid)){ tempusers in
            guard let json = tempusers else {return completion(nil)}
            print(json)
            guard let array = json.array else {return completion(nil)}
            let users = array.map{ j -> User in
                return User(jsonObject: j["user"])
            }
            completion(users)
        }
    }
    func deletePartyUser(partyid: String,userid: String, completion: @escaping (Bool) -> ()){
        task.excute(request: ServRequest.currentPartyUser(partyid: partyid)){ tempusers in
            guard let json = tempusers else {return completion(false)}
            guard let array = json.array else {return completion(false)}
            let deleteuser = array.filter{ j -> Bool in
                let user = User(jsonObject: j["user"])
                let strid = String(user.id)
                if strid == userid {
                    return true
                }else{
                    return false
                }
            }
            guard deleteuser.count == 1 else {return completion (false)}
            guard let one = deleteuser.first else {return completion(false)}
            guard let seq = one["seq"].int else {return completion (false)}
            let pid = String(seq)
            self.task.excute(request: ServRequest.deletePartyUser(partyUserId: pid)){ temp in
                guard let json = temp else {return completion(false)}
                print(json)
                completion(true)
            }
        }
    }
    func getAddressMylocationSKReverseGeocoding(place: Place,completion: @escaping (Place?) -> ()){
        task.excute(request: ServRequest.reverseGeocording(place: place)){ json in
            guard let detail = json?["addressInfo"] else{
                return completion(nil)
            }
            var place = Place(SKObject: detail)
            place.title = "내 위치"
            completion(place)
        }
    }
}

```

</p>
</details>

### MODEL
경로 데이터 구조

<details><summary>CLICK ME</summary>
<p>

##### FPMRoute.swift
```swift
//  FPMRoute.swift
//  fpm
//
//  Created by Je.vinci.Inc on 2017. 10. 23..
//  Copyright © 2017년 Crycat. All rights reserved.
//

import Foundation
import GoogleMaps
import SwiftyJSON
import Darwin

enum FEATURETYPE :String{
    case POINT = "Point"
    case LINESTRING = "LineString"
}

enum POINTTYPE: String{
    case STARTPOINT = "SP"
    case ENDPOINT = "EP"
    case GP = "GP"
}

enum TURNTYPE: Int{
    case straight = 11
    case leftturn = 12
    case rightturn = 13
    case uturn = 14
    case eightturn = 16
    case tenturn = 17
    case twoturn = 18
    case fourturn = 19
    case wp = 184
    case wp1 = 185
    case wp2 = 186
    case wp3 = 187
    case wp4 = 188
    case wp5 = 189
    case overpass = 125
    case wtf = 126
    case enterstairs = 127
    case enterrunway = 128
    case enterstairsrunway = 129
    case start = 200
    case end = 201
    case crosswalk = 211
    case leftcrosswalk = 212
    case rightcrosswalk = 213
    case eightcrosswalk = 214
    case tencrosswalk = 215
    case twocrosswalk = 216
    case fourcrosswalk = 217
    case elevator = 218
    case tempstraight = 233
}

class Feature{
    var type: FEATURETYPE
    var index: Int
    var name: String?
    var description: String?
    init(featureType: FEATURETYPE, index: Int, description: String, name: String) {
        self.type = featureType
        self.index = index
        self.description = description
        self.name = name
    }
    func returnType() -> FEATURETYPE{
        return self.type
    }
}

class Point: Feature{
    var pointtype: POINTTYPE
    var pointindex: Int
    var turntype: TURNTYPE
    var coordinate: CLLocationCoordinate2D
    var nearpoiname: String?
    var nearpoix: String?
    var nearpoiy: String?
    var intersectionname: String?
    var facilitytype: String?
    var facilityname: String?
    var direction : String?
    var Ntotaldistance: Int?
    var totaltime: Int?
    init(featureType: FEATURETYPE, index: Int, pointType: POINTTYPE, pointindex: Int, turnType: TURNTYPE,coordinate: CLLocationCoordinate2D, description: String, name: String) {
        self.pointtype = pointType
        self.pointindex = pointindex
        self.turntype = turnType
        self.coordinate = coordinate
        super.init(featureType: featureType, index: index, description: description,name: name)
    }
    init(jsonObject: JSON){
        let property = jsonObject["properties"]
        let index = property["index"].intValue
        self.pointindex = property["pointIndex"].intValue
        self.pointtype = POINTTYPE(rawValue: property["pointType"].stringValue)!
        self.turntype = TURNTYPE(rawValue: property["turnType"].intValue)!
        let geometry = jsonObject["geometry"]
        let coordinate = geometry["coordinates"].arrayValue.map({$0.doubleValue})
        self.coordinate = CLLocationCoordinate2D(latitude: coordinate[1], longitude: coordinate[0])
        let description = property["description"].stringValue
        let name = property["name"].stringValue
        super.init(featureType: .POINT, index: index, description: description,name: name)
    }
}

class LineString: Feature{
    var lineindex: Int
    var distance: Int
    var time: Int
    var coordinates: Array<CLLocationCoordinate2D> = []
    var roadtype: Int?
    var categoryroadtype: Int?
    var facilitytype: String?
    var facilityname: String?
    init(featureType: FEATURETYPE, index: Int, lineindex: Int, distance: Int, time: Int, coordinates: Array<CLLocationCoordinate2D>, description: String, name: String) {
        self.lineindex = lineindex
        self.distance = distance
        self.time = time
        self.coordinates = coordinates
        super.init(featureType: featureType, index: index,description: description,name:name)
    }
    init(jsonObject: JSON){
        let property = jsonObject["properties"]
        let geometry = jsonObject["geometry"]
        let coordinatess = geometry["coordinates"].arrayValue
        for coordinates in coordinatess{
            let tempcoordinates = coordinates.arrayValue.map({$0.doubleValue})
            let tempcoordinate = CLLocationCoordinate2D(latitude: tempcoordinates[1], longitude: tempcoordinates[0])
            self.coordinates.append(tempcoordinate)
        }
        self.lineindex = property["lineIndex"].intValue
        self.distance = property["distance"].intValue
        self.time = property["time"].intValue
        let description = property["description"].stringValue
        let name = property["name"].stringValue
        super.init(featureType: .LINESTRING, index: property["index"].intValue, description: description,name: name)
    }
}

struct EncodedLine {
    var distance: Double
    var coordinate: CLLocationCoordinate2D
}

class Direction{
    var line: Array<EncodedLine> = []
    var sequence: Int
    var path: Array<Feature> = []
    init(sequence: Int, path: Array<Feature>) {
        self.sequence = sequence
        self.path = path
    }
    func getNearLineStringCoordinateIndex(location: CLLocationCoordinate2D, lastpointIdx: Int) -> Int?{
        var inlineIdx: Int?
        let _ = path.enumerated().map{ (index,value) in
            if value.returnType() == .LINESTRING{
                let line = value as! LineString
                if line.lineindex == lastpointIdx {
                    var shorter: Double = 0
                    let _ = line.coordinates.enumerated().map{ (index, cvalue) in
                        let distance = location.getRealMeterTo(to: cvalue)
                        if shorter > distance {
                            shorter = distance
                            inlineIdx = index
                        }
                    }
                }
            }
        }
        return inlineIdx
    }
    func getNearLineIndex(location: CLLocationCoordinate2D) -> Int{
        var Idx: Int = 0
        var comparedist = self.line[0].coordinate.getRealMeterTo(to: location)
        let _ = line.enumerated().map{ (index, value) in
            let distance = value.coordinate.getRealMeterTo(to: location)
            if comparedist > distance{
                Idx = index
                comparedist = distance
            }
        }
        return Idx
    }
    func getRealMeterToLineIdx(lineIdx: Int) -> Double{
        return self.line[lineIdx].distance
    }
    func getRealMeterFromLineIdx(lineIdx: Int) -> Double{
        let count = self.line.count
        let last = self.line[count - 1].distance
        return last - self.line[lineIdx].distance
    }
    func getRealMeterFromToLineIdx(from: Int, to: Int) -> Double{
        let end = self.line[to].distance
        let start = self.line[from].distance
        return end - start
    }
    func getPointArray() -> Array<Point>{
        var pointarray: Array<Point> = []
        for i in 0..<path.count{
            if path[i].returnType() == .POINT{
                pointarray.append(path[i] as! Point)
            }
        }
        return pointarray
    }

}
extension CLLocationCoordinate2D{
    func getRealMeterTo(to: CLLocationCoordinate2D) -> Double{
        let startcl = CLLocation(latitude: self.latitude, longitude: self.longitude)
        let endcl = CLLocation(latitude: to.latitude, longitude: to.longitude)
        return endcl.distance(from: startcl)
    }
}

extension LineString{
    func returnArrayCoordinate() -> Array<CLLocationCoordinate2D>{
        return self.coordinates
    }
}

extension Direction{
    func returnArrayCoordinate() -> Array<CLLocationCoordinate2D>{
        var result: Array<CLLocationCoordinate2D> = []
        for i in 0..<self.path.count{
            if path[i].returnType() == .LINESTRING{
                let linestring = path[i] as! LineString
                if i >= 2{
                    var line = linestring.returnArrayCoordinate()
                    line.remove(at: 0)
                    result.append(contentsOf: line)
                }else{
                    result.append(contentsOf: linestring.returnArrayCoordinate())
                }

            }
        }
        for i in 0..<result.count{
            if i == 0{
                let line = EncodedLine(distance: 0, coordinate: result[0])
                self.line.append(line)
            }else {
                let distance = result[i - 1].getRealMeterTo(to: result[i]) + self.line[i - 1].distance
                let line = EncodedLine(distance: distance, coordinate: result[i])
                self.line.append(line)
            }

        }
        return result
    }
    func returnArrayCoordinateFrom(lineIdx: Int, inlineIdx: Int) -> Array<CLLocationCoordinate2D>{
        var firstcoordinates: Array<CLLocationCoordinate2D> = []
        return []
    }
    func returnArrayCoordinateTo(lineIdx: Int, inlineIdx: Int) -> Array<CLLocationCoordinate2D>{
        return []
    }
}

extension FPMData{
    func returnArrayCoordinate() -> Array<CLLocationCoordinate2D>{
        var result: Array<CLLocationCoordinate2D> = []
        for i in 0..<self.directions.count{
            result.append(contentsOf: directions[i].returnArrayCoordinate())
        }
        return result
    }
}

extension GMSMutablePath{
    class func GMSMutablePathwithArrayCoordi(arraycoordinate: Array<CLLocationCoordinate2D>) -> GMSMutablePath{
        let path = GMSMutablePath()
        for i in 0..<arraycoordinate.count{
            path.add(arraycoordinate[i])
        }
        return path
    }
}
```

</p>
</details>
받은 장소 데이터를 T Map 로 변환하여 T Map API를 통해 경로 데이터를 받아옵니다.


### [LocalNotification, Circle을 통한 위치 확인 따른 네비게이션 안내]
### [UserData저장기능, MinimapCapture]
### [WebSocket을 통한 실시간 위치공유, Firebase를 통한 채팅, 파티 생성시 사용되는 APNS ]
### [WebSocket 채팅 코드내용]
